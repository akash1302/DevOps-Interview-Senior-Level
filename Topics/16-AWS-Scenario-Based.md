# Senior DevOps Interview Questions: AWS Scenario-Based

### Q: You are tasked with designing a scalable web application on AWS to handle fluctuating traffic. What services and architecture would you use to ensure both availability and cost-efficiency?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd start with a multi-AZ architecture, not a single big server.

For the application layer, I'd use an **ALB** in public subnets, running the app in **ECS Fargate or EKS** in private subnets across at least two AZs. The ALB spreads traffic across them and drops any AZ that goes unhealthy.

For the database, **RDS/Aurora Multi-AZ**, and I'd add **ElastiCache** in front of it for anything read-heavy so the database isn't taking the full hit on every request.

Autoscaling is based on CPU, memory, or request count — more tasks come up automatically during a spike, and scale back down once traffic drops, so I'm not paying for peak capacity around the clock.

Static content goes through **S3 + CloudFront**, so it's not even hitting the app servers. CloudWatch handles monitoring and alarms.

**Simple flow:** User → Route 53 → CloudFront/WAF → ALB → ECS/EKS → RDS/Aurora (+ ElastiCache)

**Key point:** I don't just throw bigger servers at it. I make the app horizontally scalable so it absorbs spikes without keeping expensive idle capacity running the rest of the time.

</details>

---

### Q: Your application needs to process large volumes of data and convert it into various formats for downstream services. What AWS services would you use to automate and scale this process?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I land the raw files in an **S3** bucket first, since that gives me durable storage and a natural event trigger point.

An S3 event triggers processing. For lightweight, fast conversions, that's a **Lambda** function straight off the event. For heavier jobs — large files, CPU-intensive conversion, anything that could run longer than Lambda's timeout — I use **AWS Batch** instead, since it can run on right-sized compute and isn't time-boxed the way Lambda is.

If the pipeline has multiple steps — validate, convert, then notify a downstream service — I use **Step Functions** to orchestrate it, so each stage's success/failure is tracked properly instead of chaining Lambdas together by hand with custom retry logic.

For decoupling from downstream consumers, converted output goes to a second S3 bucket, and an **SQS** queue notifies whatever service needs to pick it up, so a slow downstream consumer doesn't block the pipeline.

**Simple flow:** Raw file → S3 → event trigger → Lambda (light) or Batch (heavy) → Step Functions orchestrates multi-step jobs → output S3 → SQS notifies downstream.

**Key point:** I split by workload size — Lambda for fast and small, Batch for heavy and long-running — instead of forcing everything through one tool.

</details>

---

### Q: Your organization has decided to use AWS for disaster recovery but wants to minimize costs. What disaster recovery architecture would you recommend?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The actual architecture depends entirely on the RTO and RPO the business is willing to state — I never design DR before getting those two numbers first.

If the business can tolerate a few hours of downtime, I'd go with **pilot light** — core infrastructure like the database is replicated continuously (RDS cross-region read replica, or S3 cross-region replication for data), but the compute layer, ALB, and app servers stay switched off until an actual disaster, then get spun up from an AMI or IaC template. This keeps ongoing cost low since you're mostly just paying for storage and replication, not idle compute.

If they need faster recovery, **warm standby** runs a smaller version of the full stack at all times in the second region, ready to scale up on failover, which costs more than pilot light but recovers in minutes instead of the time it takes to boot everything from scratch.

I'd avoid a full active-active hot standby unless the RTO is genuinely near-zero, since running full duplicate capacity 24/7 is the most expensive option and usually isn't justified by the actual business requirement.

**Simple flow:** Primary region running live → data replicated continuously to DR region (RDS replica / S3 CRR) → compute stays off or minimal in DR → Route 53 failover triggers on health check → compute scales up in DR region.

**Key point:** Cost and recovery time are a direct trade-off — I match the architecture to the real RTO/RPO instead of defaulting to the most robust (and most expensive) option.

</details>

---

### Q: You need to ensure secure and compliant data transfer between your on-premises data center and AWS. What would you recommend?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For anything beyond occasional small transfers, I'd recommend **AWS Direct Connect** — it's a dedicated physical link, so it bypasses the public internet entirely, giving consistent bandwidth and lower latency, which also matters for compliance since the traffic isn't traversing shared public infrastructure.

For encryption on top of that link, I'd run a **VPN over Direct Connect** (or MACsec for private VIFs on newer Direct Connect connections) — Direct Connect alone gives you a private path, but it's not encrypted by default, and a lot of compliance frameworks specifically require encryption in transit regardless of whether the path is private.

If Direct Connect isn't justified yet — lower volume, or it's still being provisioned, which can take weeks — I'd use a **Site-to-Site VPN** with IPsec as either the primary path or a fallback, since it's encrypted by default and quick to stand up.

For file-based transfers specifically, like SFTP workflows from legacy on-prem systems, I'd use **AWS Transfer Family** instead of building custom infrastructure, since it gives a managed, auditable SFTP endpoint that lands directly in S3.

**Key point:** Direct Connect for the dedicated path, encryption layered on top (VPN or MACsec) for compliance, VPN alone as a faster-to-provision fallback or for lower-volume needs.

</details>

---

### Q: You are managing a microservices-based application that is experiencing latency due to high database load. How would you optimize the architecture?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't jump straight to a bigger database instance — I check what's actually generating the load first, since scaling compute doesn't fix a genuinely bad query or a missing index.

If it's read-heavy load, I add **ElastiCache** in front of the database for frequently-read data, and **RDS read replicas** for queries that need to be reasonably fresh but not necessarily the absolute latest write — reporting queries are a common one to offload this way.

I also check connection handling — a lot of microservices setups hit connection exhaustion under load because each service instance holds its own pool, and they add up fast. I'd put **RDS Proxy** in front of the database so hundreds of app connections share a much smaller, pooled set of actual database connections.

For write-heavy operations that don't need to happen synchronously in the request path — like sending a notification or updating an analytics table after an order is placed — I move those to **SQS** so the request returns fast and the actual database write happens asynchronously in the background.

**Simple flow:** Identify read vs write vs connection-exhaustion bottleneck → reads: ElastiCache + read replicas → connections: RDS Proxy → non-critical-path writes: async via SQS.

**Key point:** I fix the actual bottleneck — reads, connections, or synchronous writes — rather than reflexively resizing the database instance.

</details>

---

### Q: You have a critical application that cannot tolerate any downtime. How would you architect this solution on AWS to achieve high availability?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

"Cannot tolerate any downtime" gets multi-AZ as the baseline, non-negotiable — app servers across at least two AZs behind an ALB, and **Aurora Multi-AZ** for the database so a single AZ failure doesn't take the app down.

If the tolerance is truly zero, not just "very low," I'd go further to **multi-region active-active** — Aurora Global Database for cross-region replication with fast promotion, app deployed in two regions, and **Route 53** with health-check-based failover routing so traffic shifts automatically if an entire region has a problem, not just an AZ.

Deployments themselves also have to not introduce downtime — I'd use blue-green or rolling deployments through **CodeDeploy** or an ECS/EKS-native rolling update, never a straight swap where old and new both go down at once.

I'd also make sure health checks are genuinely meaningful — checking that the app can actually reach its database and dependencies, not just that the process is running — since a shallow health check will happily route traffic to an instance that's technically up but functionally broken.

**Key point:** True zero-downtime is a chain — multi-AZ or multi-region infrastructure, automated failover, and deployments that never take capacity below 100% at any point.

</details>

---

### Q: Your application requires secure storage of sensitive customer data, with auditing and monitoring of all access attempts. What would you do?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Data at rest gets encrypted with **SSE-KMS**, using a customer-managed key rather than the default AWS-managed key, since a customer-managed key gives me control over rotation and lets me see exactly who used it via CloudTrail.

Access is locked down with a tight **bucket policy** plus **IAM policies** scoped to exactly what each role needs — no wildcard `s3:*` on sensitive buckets. I'd also add a bucket policy condition that denies any request not coming over TLS, so encryption in transit is enforced, not just assumed.

For auditing, I turn on **CloudTrail data events** specifically for that bucket — management events are on by default, but object-level read/write access needs data events explicitly enabled, and that's the actual audit trail that shows who read a specific object and when.

On top of that, **GuardDuty** and **Macie** — GuardDuty flags anomalous access patterns, like an access key suddenly reading from an unusual location, and Macie scans the bucket to confirm sensitive data isn't sitting somewhere it shouldn't, or exposed more broadly than intended.

**Key point:** Encryption alone isn't the control — the real requirement here is the audit trail, so CloudTrail data events on that specific bucket are the piece I never skip.

</details>

---

### Q: Your team needs a continuous integration/continuous delivery (CI/CD) pipeline to automatically deploy applications in AWS. How would you set it up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd wire it as: source triggers build, build produces an artifact, artifact gets deployed — whether that's native AWS tooling or GitHub Actions/GitLab CI, the shape is the same.

Source is Git-based, triggering on a PR merge to the target branch. Build runs in **CodeBuild** (or the CI platform's own runners) — install, test, build the container image, push to **ECR**. I always run the security scan and unit tests at this stage, before anything gets close to deployment, so a bad build fails fast and cheap.

Deployment goes through **CodeDeploy** for ECS/EC2, doing a blue-green or rolling rollout, or through an ArgoCD-style GitOps flow if it's EKS. Either way, production is gated behind manual approval, never auto-deployed straight from a merge.

For the AWS access itself, the pipeline authenticates via **OIDC** to assume a short-lived, tightly-scoped IAM role — no long-lived access keys stored in the CI system.

**Simple flow:** PR merges → CodeBuild builds & tests → image pushed to ECR → CodeDeploy/ArgoCD deploys → manual approval gate before production → OIDC-based short-lived credentials throughout.

**Key point:** Production is always gated behind human approval, and the pipeline never holds a long-lived AWS credential.

</details>

---

### Q: Your web application experiences a large number of requests from a particular geographical region causing latency. How would you improve performance for users in that region?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, I confirm it's actually distance-driven latency and not a backend bottleneck — checking CloudFront/ALB metrics by region tells me quickly whether it's genuinely the physical distance to the origin or something else entirely.

If it is distance, **CloudFront** in front of the app is the first move — its edge locations cache static and cacheable content close to users, so a big chunk of requests never have to reach the origin region at all.

For requests that do need to hit the backend — dynamic, non-cacheable ones — I'd consider actually deploying a regional stack in or near that geography if the traffic volume justifies it, with **Route 53 latency-based routing** sending users to whichever region is actually fastest for them.

For anything involving large file uploads specifically, like user-submitted media, **S3 Transfer Acceleration** routes the upload through the nearest CloudFront edge instead of going directly to the bucket's home region over the public path, which noticeably helps for users far from that region.

**Key point:** CDN first for anything cacheable, since it's the cheapest and fastest fix — a full regional deployment is the next step up only if dynamic traffic volume from that region actually justifies the cost.

</details>

---

### Q: You need to implement fine-grained access control for a growing number of users accessing an AWS S3 bucket. How would you design this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't manage this with individual IAM users — that doesn't scale past a handful of people. I use IAM roles tied to groups (or federated through IAM Identity Center if it's tied to the company's SSO), so access is managed by group membership, not per-person policy edits.

For actual fine-grained control — different teams needing access to different prefixes within the same bucket — I use **S3 Access Points** scoped per team or use case, each with its own policy, rather than one giant bucket policy trying to express every team's rules in one place. That also makes it much easier to reason about and audit who can touch what.

Policies are scoped down to the specific prefix and action a role actually needs — `s3:GetObject` on `team-a/*` for team A, nothing broader — and I use policy conditions where it matters, like restricting access to a specific VPC endpoint so the bucket can't be reached from outside the company network at all.

As the user count grows, I review access on a schedule, not just when someone happens to ask — permissions accumulate over time as people change roles, and nobody proactively removes the old ones unless it's a standing process.

**Key point:** Access points plus prefix-scoped, condition-restricted IAM policies — not one broad bucket policy trying to cover every team's access pattern.

</details>

---

### Q: Your application is built on containers, and you need a solution that allows automatic scaling of containers without managing underlying servers. What AWS service would you recommend?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**ECS with Fargate** is what I'd default to here — it's exactly built for this, containers run without me provisioning or patching any EC2 instances underneath them at all.

Autoscaling is based on CPU, memory, or a custom CloudWatch metric like request count per target, and Fargate just launches more tasks to match, without me managing a node group's capacity the way I would on EC2-backed ECS or a self-managed EKS cluster.

If the team is already committed to Kubernetes specifically — for portability, or existing tooling built around the Kubernetes API — **EKS with Fargate profiles** gives the same "no server management" benefit while staying on Kubernetes, though I'd flag that Fargate on EKS has some real constraints, like no DaemonSets and no privileged containers, so it's not a drop-in replacement for every EKS workload.

For most teams not already deep into Kubernetes tooling, I'd steer toward plain ECS Fargate — it's simpler to operate day to day and there's less platform overhead to maintain.

**Key point:** ECS Fargate is the default answer for "containers, autoscaling, no server management" — EKS Fargate only if there's a real, existing reason the team needs Kubernetes specifically.

</details>

---

### Q: You need to archive data that is rarely accessed but must be retained for compliance reasons. How would you store this data in AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**S3 Glacier** or **Glacier Deep Archive**, depending on how rarely it's actually accessed and how fast retrieval needs to be if it ever is.

I don't manually move data there — I set up an **S3 Lifecycle policy** on the source bucket, so objects transition automatically after a defined age, like moving to Glacier after 90 days and Deep Archive after a year. That way nobody has to remember to do it, and it happens consistently across every object, not just the ones someone thought to move by hand.

If retrieval time actually matters — some compliance audits do need data back within hours, not the 12+ hours Deep Archive can take — I'd use regular Glacier with expedited retrieval instead, which costs more per retrieval but gets data back in minutes when genuinely needed.

For compliance specifically, I also enable **S3 Object Lock** in compliance mode on that bucket, so data genuinely cannot be deleted or overwritten before the retention period ends, by anyone, including the account root — that's often the actual requirement compliance is asking for, not just "cheap storage."

**Key point:** Lifecycle policy for automatic tiering, Object Lock in compliance mode for the actual retention guarantee auditors are checking for.

</details>

---

### Q: You need to optimize costs for a fleet of EC2 instances that run non-critical workloads during business hours. How can you achieve cost savings without sacrificing performance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If these only need to run during business hours, the biggest single win is just not running them the rest of the time — I'd set up a scheduled stop/start using **EventBridge scheduled rules triggering a Lambda**, or the AWS Instance Scheduler solution if it's a larger fleet, so instances shut down every evening and weekend and start back up before the business day.

That alone is often a 60-70% reduction on that fleet's compute cost, since roughly two-thirds of the week is outside business hours, and it doesn't touch performance at all during the hours anyone's actually using them.

On top of the schedule, I'd right-size the instances — checking real CPU and memory usage against what's provisioned, since these are non-critical workloads and are often over-provisioned from an initial rough guess that nobody revisited.

If the workload can tolerate interruption, I'd also consider **Spot Instances** for the business-hours window itself, since non-critical is usually a reasonable signal that occasional interruption is acceptable, stacking further savings on top of the scheduling.

**Key point:** Scheduled shutdown outside business hours is the biggest lever by far — right-sizing and Spot are the next layer on top of that, not a replacement for it.

</details>

---

### Q: Your application handles sensitive data, and you need to ensure encryption of data in transit and at rest. How would you architect this solution on AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For data at rest, I enable **KMS encryption** across every storage layer that touches the data — S3 with SSE-KMS, RDS/EBS with KMS-backed encryption at the volume or instance level. I use a customer-managed key rather than the AWS-managed default so I control rotation and can see key usage in CloudTrail.

For data in transit, TLS is enforced end to end — **ACM** issues and manages the certificate on the ALB, so the app doesn't have to handle certificate renewal manually, and I set the ALB listener to redirect any plain HTTP request to HTTPS rather than silently accepting it.

Internally, between services, I also enforce TLS for service-to-service traffic where it carries sensitive data, not just at the public-facing edge — it's a common gap to encrypt the outside and assume internal VPC traffic is automatically safe.

To make sure this isn't just a one-time setup, I enforce it structurally — an S3 bucket policy that denies any request not made over TLS, and for a stricter environment, an SCP at the AWS Organizations level that blocks creating unencrypted resources in the first place, so it's not dependent on every engineer remembering to tick the encryption box.

**Key point:** Encryption enforced structurally, through policy conditions and SCPs, not just configured once and trusted to stay that way.

</details>

---

### Q: You need to migrate a large database from your on-premises environment to AWS with minimal downtime. How would you achieve this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**AWS DMS** with continuous replication (CDC) is what I'd use here, not a one-shot dump-and-restore, since a straight export/import on a large database means real downtime for however long that transfer takes.

DMS does an initial full load of the existing data into the target RDS/Aurora instance, and once that's done, it switches to change data capture, continuously replicating new writes from the on-prem database as they happen, so the target stays in sync while the source database keeps running normally the whole time.

Cutover happens during a short, planned maintenance window — I stop writes to the source, let DMS catch up the last few seconds or minutes of changes, point the application at the new AWS database, and resume traffic. That window is usually minutes, not the hours a full migration would otherwise take.

I always test the actual cutover process against a staging copy first, and I validate data consistency between source and target with a checksum or row-count comparison before the real cutover, not after — finding a data mismatch after production traffic has already moved is a much worse day than catching it in a rehearsal.

**Simple flow:** DMS full load → CDC keeps target in sync with ongoing writes → short maintenance window → final catch-up → cutover app to new database → validate.

**Key point:** CDC-based replication is what turns a normally multi-hour migration into a cutover window measured in minutes.

</details>

---
