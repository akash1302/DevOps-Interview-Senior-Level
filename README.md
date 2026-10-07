<div align="center">

# DevOps Interview Q&A — Senior / Principal Level

![Topics](https://img.shields.io/badge/topics-14-blue)
![Level](https://img.shields.io/badge/level-Senior--Principal-orange)
![Format](https://img.shields.io/badge/format-Markdown-informational)
![Stack](https://img.shields.io/badge/stack-AWS%20%7C%20K8s%20%7C%20Terraform%20%7C%20CI%2FCD-success)

Simple, natural, first-person candidate answers — the way an experienced engineer actually talks in an interview.

</div>

---

## 📁 Repository Structure

This README covers 14 core domains as one continuous reference (table of contents below). Beyond that, the repo has two more layers of prep material:

| # | Topic | File |
|---|-------|------|
| 01 | AWS | [`Topics/01-AWS.md`](./Topics/01-AWS.md) |
| 02 | Terraform | [`Topics/02-Terraform.md`](./Topics/02-Terraform.md) |
| 03 | Docker | [`Topics/03-Docker.md`](./Topics/03-Docker.md) |
| 04 | Kubernetes / EKS | [`Topics/04-Kubernetes-EKS.md`](./Topics/04-Kubernetes-EKS.md) |
| 05 | CI/CD | [`Topics/05-CICD.md`](./Topics/05-CICD.md) |
| 06 | Linux | [`Topics/06-Linux.md`](./Topics/06-Linux.md) |
| 07 | Networking | [`Topics/07-Networking.md`](./Topics/07-Networking.md) |
| 08 | Security | [`Topics/08-Security.md`](./Topics/08-Security.md) |
| 09 | Monitoring & Logging | [`Topics/09-Monitoring-Logging.md`](./Topics/09-Monitoring-Logging.md) |
| 10 | Production Troubleshooting | [`Topics/10-Production-Troubleshooting.md`](./Topics/10-Production-Troubleshooting.md) |
| 11 | DevOps Architecture | [`Topics/11-DevOps-Architecture.md`](./Topics/11-DevOps-Architecture.md) |
| 12 | Real-World Scenarios | [`Topics/12-Real-World-Scenarios.md`](./Topics/12-Real-World-Scenarios.md) |
| 13 | Git & GitHub/GitLab | [`Topics/13-Git-GitHub-GitLab.md`](./Topics/13-Git-GitHub-GitLab.md) |
| 14 | Database & Caching | [`Topics/14-Database-Caching.md`](./Topics/14-Database-Caching.md) |
| 15 | Cost Optimization / FinOps | [`Topics/15-Cost-Optimization-FinOps.md`](./Topics/15-Cost-Optimization-FinOps.md) |
| 16 | AWS Scenario-Based | [`Topics/16-AWS-Scenario-Based.md`](./Topics/16-AWS-Scenario-Based.md) |
| 17 | Naggarow Interview Senior Level | [`17-naggarow.md`](./Topics/17-naggarow.md) |

Each topic file uses the same style of answers as this README.

| Day | Focus | File |
|-----|-------|------|
| Day 1 | AWS VPC & Networking (production troubleshooting) | [`DevOps/DAY-1-AWS-VPC.md`](./DevOps/DAY-1-AWS-VPC.md) |
| Day 2 | Kubernetes, Docker, CI/CD, AWS, SRE — principal-level deep dives | [`DevOps/DAY-2-PRINCIPAL-DEEPDIVE.md`](./DevOps/DAY-2-PRINCIPAL-DEEPDIVE.md) |
| Day 3 | Terraform scenario-based questions | [`DevOps/DAY-3-TERRAFORM-SCENARIOS.md`](./DevOps/DAY-3-TERRAFORM-SCENARIOS.md) |

---

## Table of Contents

| # | Domain | # | Domain |
|---|--------|---|--------|
| 1 | [Docker — Container Internals](#1-docker--container-internals) | 8 | [AWS Infrastructure & Networking](#8-aws-infrastructure--networking) |
| 2 | [Kubernetes — Orchestration & Control Plane](#2-kubernetes--orchestration--control-plane) | 9 | [ECS Fargate + ALB — Production Failure Modes](#9-ecs-fargate--alb--production-failure-modes) |
| 3 | [CI/CD — Jenkins & Pipeline Engineering](#3-cicd--jenkins--pipeline-engineering) | 10 | [IAM, Secrets & Security Boundaries](#10-iam-secrets--security-boundaries) |
| 4 | [Git — Version Control Internals](#4-git--version-control-internals) | 11 | [Serverless — Lambda Architecture](#11-serverless--lambda-architecture) |
| 5 | [Terraform — State & Fundamentals](#5-terraform--state--fundamentals) | 12 | [Observability — Monitoring, Logging & Auditing](#12-observability--monitoring-logging--auditing) |
| 6 | [Terraform — Multi-Account Architecture](#6-terraform--multi-account-architecture) | 13 | [Linux / Scripting — Systems Operations](#13-linux--scripting--systems-operations) |
| 7 | [Terraform — Drift, Recovery & Advanced Ops](#7-terraform--drift-recovery--advanced-ops) | 14 | [System Design — End-to-End Architecture](#14-system-design--end-to-end-architecture) |

---

## 1. Docker — Container Internals

### Q: What are the volume types in Docker, and what are the performance/security implications of each?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In my experience, there are three types I deal with. A **bind mount** maps a folder from the host straight into the container — it's fast, but the container can now see real host files, and I've had permission issues because the container's user ID didn't match the host folder's owner.

A **named volume** is managed fully by Docker, so it's safer, and it doesn't get wiped by a normal cleanup command. This is what I use for anything stateful, like a database.

An **anonymous volume** is similar but has no name, so it's easy to forget and it can quietly use up disk space over time.

So my rule is simple: named volumes for anything I need to keep, and I check for leftover anonymous volumes on the host every so often.

</details>

---

### Q: `CMD` vs `ENTRYPOINT` — what's the actual execution model, and when does mixing them break a container's signal handling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`ENTRYPOINT` is the main process that runs. `CMD` just gives it default arguments.

What I normally check for is the format. If someone writes `CMD npm start` as plain text instead of the array form, Docker runs it inside a shell, and that shell becomes the main process instead of the app itself. So when Docker tries to stop the container, the app never gets the stop signal, and the container just hangs until it's force-killed.

For example, I had exactly this issue once — the container looked fine in testing but never shut down cleanly in production. The fix was switching to the array format, like `ENTRYPOINT ["node", "server.js"]`, so the app itself receives the signal directly.

</details>

---

### Q: A container restarts and all data is gone. What's actually happening at the storage-driver level, and how do you architect around it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This isn't really a bug, it's expected behavior once you understand it. Every container has its own writable space, and that space gets removed when the container itself is removed, not just stopped.

So if the app writes data to a path with no volume attached, that data was never going to survive a restart in the first place.

What I normally do is mount a real volume, or a PersistentVolumeClaim in Kubernetes, at every path the app actually needs to keep. I don't assume I know every path — I check the running container to see exactly where it's writing before trusting the setup.

</details>

---

### Q: How do you reclaim disk on a Docker host without risking an active build cache or in-use image?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never run a full cleanup command blindly, especially on a CI server. I've seen it wipe out a build cache another job still needed, which just made builds slower for no reason.

What I normally do instead is clean up in a targeted way — only images older than a few days, so recent work is never touched.

For example, on CI runners, I keep the build cache on its own separate volume, so a cleanup job can't accidentally remove it. I also keep an eye on disk usage as a regular check, so I catch it early instead of finding out when a build fails.

</details>

---

## 2. Kubernetes — Orchestration & Control Plane

### Q: What do taints and tolerations actually enforce at the scheduler level, and what do they *not* protect against?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A taint on a node stops pods from being scheduled there. A toleration on a pod just allows it to get past that block — it doesn't force the pod onto that node.

This only controls scheduling, it's not a security boundary. A pod that's already running, or one placed a different way, isn't affected by a taint at all.

In my experience, if I actually need dedicated nodes, I pair the taint with node affinity as well, so the pod is required to go there, not just allowed. If I need real isolation, I add a NetworkPolicy on top of that too.

</details>

---

### Q: Is pod-to-pod traffic allowed by default, and how do you enforce a default-deny posture without breaking DNS or control-plane traffic?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Yes, by default any pod can talk to any other pod in the cluster. There's no restriction unless you add one.

What I normally see happen the first time someone turns on a deny-all policy is DNS breaks immediately, because the DNS service sits in a different namespace and needs its own explicit rule to still be allowed.

So I always test this in a non-production namespace first, and I add a clear rule allowing DNS traffic as part of the same change, not as a fix afterward.

</details>

---

### Q: `StatefulSet` vs `Deployment` — what specific guarantees does a `StatefulSet` provide that make it non-optional for stateful workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Deployment is fine when any pod can replace any other pod, like a stateless web server.

For something like a database, I use a StatefulSet, because it gives each pod a fixed name and its own dedicated storage that follows it around, even if it moves to a different node.

In my experience, running a database on a plain Deployment looks fine at first, but breaks during the first rolling update, since pod identity and storage aren't guaranteed to stay matched. That's exactly why I treat StatefulSet as required for anything stateful, not optional.

</details>

---

### Q: A pod is stuck in `CrashLoopBackOff`. Walk through the exact diagnostic sequence and what each signal tells you.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I check `kubectl describe pod` for the exit code and recent events — that tells me what actually happened, not just that it failed.

Then I check `kubectl logs --previous`, since the current instance already restarted, and the previous instance's logs are what actually show the real error.

If the exit code is 137, that's almost always the app running out of memory, so I check the memory limit against real usage. Any other exit code usually points to a bug or bad config in the app itself, not the platform.

</details>

---

### Q: An application deployed to EKS isn't reachable externally. What's the exact layer-by-layer isolation boundary you check, in order?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check this in a fixed order, so I don't waste time chasing the wrong thing.

First, whether the Service actually has a healthy pod behind it. Then the port setting, then the load balancer or ingress, then the security groups between the load balancer and the nodes, and finally DNS.

In my experience, the security group step is where I find the problem most often on EKS specifically — the load balancer's security group and the node's security group are two separate things, and people usually only check one of them.

</details>

---

### Q: How does cross-pod communication actually traverse the stack inside an EKS cluster, mechanically?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In EKS, every pod gets a real IP address from the VPC itself, not some separate fake network. That's actually why the number of pods per node is limited — it depends on how many IPs that instance type can hand out.

When a pod calls another service by name, it goes through the cluster's DNS to find the address first, and then gets routed to the actual pod behind the scenes.

For a cluster with a lot of traffic, I'd check whether the routing mode is actually set up to handle that scale well, since the default setup can start to slow down once you have a lot of services.

</details>

---

## 3. CI/CD — Jenkins & Pipeline Engineering

### Q: What are the trade-offs between Jenkins deployment models (static EC2, Docker, Helm-on-EKS), and which failure modes does each eliminate or introduce?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I've run Jenkins all three ways at different points.

A plain server install is simple, but it's a single point of failure, and how many builds can run at once is limited by that one server. Running it in Docker helps a little, but builds can still leave leftover files behind for the next job.

What I run now is Jenkins on Kubernetes — every build gets its own fresh worker that gets thrown away right after, so nothing carries over between builds, and capacity grows with the cluster automatically instead of being stuck at one fixed size.

</details>

---

### Q: A Jenkins pipeline fails intermittently, not on every run. What's the deterministic debugging sequence, and how do you distinguish a flaky test from an infrastructure race condition?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't call something flaky until I've actually checked, since that label gets used too easily.

First I check if the failures line up with new build workers starting up — if they do, that's a timing issue with the infrastructure, not the test. Then I try rerunning with everything set to run one at a time. If the failure goes away, it was a resource conflict, not the test itself.

Only after ruling both of those out do I actually mark it as a flaky test and track it properly, rather than just adding a retry and moving on.

</details>

---

### Q: How do you manage Jenkins plugin upgrades on a Helm-deployed instance without an untested plugin breaking the production pipeline?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never update plugins by clicking around in the Jenkins UI, since on a Kubernetes setup those changes get overwritten the next time it's deployed anyway.

Plugin versions are set in one config file, and that's the real source of truth. Before bumping a version, I test it on a separate non-production Jenkins first, using our actual pipelines, not a sample one.

When I do roll it out, I use a deployment method that automatically rolls back if the health check fails, so I never end up stuck with a half-updated instance.

</details>

---

### Q: The Jenkins admin credential is lost and the controller runs on Kubernetes with no external secret backup. What's the recovery path, and how do you prevent this from being a single point of failure again?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

It depends on where this happens. If it's still the first-time setup, the starting password is in a file I can just read directly from the pod.

If it's a real admin account lost after setup, I have to get into the server and reset it manually, which is not something you want to be figuring out for the first time during an actual incident.

Going forward, the real fix is connecting Jenkins to the company's normal login system, so there's no single password to lose. I also make sure the Jenkins data is backed up on a schedule, just as a safety net.

</details>

---

### Q: Design a rollback strategy set that covers every layer of a deployment — application, infrastructure, and pipeline state.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Rollback isn't one thing, each layer needs its own plan.

For the app in Kubernetes, there's a direct command to undo the last change instantly, no rebuild needed. For Terraform, there's no real rollback command, so reviewing the plan carefully before applying is the actual safety net.

For source code, I always use a safe revert, never a hard reset on a shared branch, since that rewrites history everyone else already has.

</details>

---

## 4. Git — Version Control Internals

### Q: Explain the branching model you enforce, and specifically why `merge` vs `rebase` changes the risk profile of a shared branch.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Merge just adds a new commit and keeps the real history, so it's always safe on a shared branch.

Rebase rewrites history with new commit IDs, so doing it on a branch other people already have breaks their copy too.

So my rule is simple — I only rebase my own branch before I push it, never after it's shared. Day to day, I prefer short-lived branches that get merged into main quickly, rather than long-lived branches that drift and cause painful conflicts later.

</details>

---

### Q: How do you cleanly squash a range of commits before merging, and what breaks if you do it after pushing to a shared branch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use an interactive rebase to combine a few messy commits into one clean commit.

The problem is if that branch is already shared, this creates new commit IDs and breaks everyone else's copy. So I always force-push using the safer option that protects against overwriting someone else's work, not a plain force push.

And if the branch really is already shared, I give the team a heads-up before I touch it.

</details>

---

### Q: The `.git` directory is gone from a working copy. What's actually recoverable, and what isn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The `.git` folder holds the whole project history, so losing it means losing anything that was never pushed anywhere else.

If there's a remote and everything was already pushed, it's easy to fix — just get a fresh copy, nothing's actually lost. But any local commit that was never pushed is gone for good.

That's exactly why I push often, and I don't let real work sit only on my own machine for too long.

</details>

---

### Q: `git fetch` vs `git pull` — what's the actual difference in terms of working-tree risk?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git fetch` just downloads the latest changes, it doesn't touch my own files at all, so it's completely safe.

`git pull` downloads and then automatically merges it into my current work, and if I have local changes that conflict, that can leave a mess.

I use pull for everyday work, but I never let it run inside a script or pipeline without a setting that stops it from silently merging something unexpected.

</details>

---

## 5. Terraform — State & Fundamentals

### Q: How do you bring an out-of-band-created AWS resource under Terraform management without recreating it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is `terraform import`, but it only updates the state file — it doesn't write the matching code for me.

So what I actually do is write the matching resource block first, run the import, and then run `terraform plan` right away to check nothing looks different.

For example, I've seen people skip that last check and end up with Terraform trying to delete the exact resource it just imported, because the code didn't fully match reality.

</details>

---

### Q: What specifically breaks when Terraform state is kept local in a multi-engineer team, beyond "it's not backed up"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The real problem is there's no locking. If two people run `apply` at the same time, one silently overwrites the other's changes, and Terraform ends up not even knowing some real resources exist anymore.

In my experience, a remote backend with locking, like S3 with a lock table, isn't optional for a team — it's needed from day one. I also turn on versioning on that storage, since that's the real way to recover a bad state file.

</details>

---

### Q: What does a real Terraform testing/validation pipeline enforce before `apply` ever runs against production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform validate` only checks syntax, it has no idea if something's actually valid on AWS's side.

So what I normally set up is a few checks in sequence — a format check, then validate, then a linter for real provider-level mistakes, then a security scan, and then a plan that a real person reviews before anything is applied.

The goal is catching mistakes as early as possible, not at apply time when it's already touching real infrastructure.

</details>

---

## 6. Terraform — Multi-Account Architecture

### Q: Design the module/state topology for a Terraform codebase spanning dev/staging/prod across separate AWS accounts.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build shared, reusable modules for things like a VPC or a database, with no environment-specific logic inside them.

Then each environment gets its own small folder that calls those modules with its own settings, and its own separate state file tied to its own account.

CI switches into the correct account before it ever touches anything real, so there's no shared state that could accidentally get applied to the wrong environment.

</details>

---

### Q: How do you prevent a Terraform state read/write in one AWS account from ever touching another account's state, structurally rather than by convention?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't rely on people remembering to point at the right config, since that eventually breaks.

Instead, each account gets its own storage bucket for state, and the access policy only allows that account's own pipeline to touch it. So even a mistake in a pipeline setting physically can't reach another account's state.

That's a real, structural wall, not just a rule written in a document somewhere.

</details>

---

### Q: How do you keep secrets (DB credentials, API keys) out of both the Terraform codebase and the state file, given that Terraform state stores resource attributes in plaintext by default?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Something people miss is that Terraform's state file is plain text by default, so even a secret passed in as a variable ends up sitting in that file.

My rule is simple — secrets never go in as raw variables. They live in a proper secrets manager, and Terraform just references them.

On top of that, I turn on encryption on the state storage itself, and I make sure only the people or pipelines who really need it can read it.

</details>

---

### Q: Trace exactly what happens end-to-end when a Terraform change merges to `main` in a CI/CD-driven multi-account pipeline.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, Terraform connects to the right backend for that environment, then builds a plan, and that plan gets posted somewhere a person can actually review it.

After manual approval, specifically for production, the pipeline switches into that account and applies that exact same plan, not a fresh one.

Every log and plan output gets kept, tied back to the commit that started it, so there's a clear audit trail if something needs to be checked later.

</details>

---

## 7. Terraform — Drift, Recovery & Advanced Ops

### Q: Someone manually changes a resource in the AWS console that Terraform manages. What does `terraform plan` actually show you, and how do you resolve it correctly versus incorrectly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform plan` will show a change that puts things back to what's in the code — that's expected, since Terraform doesn't know who made a change, it just compares.

The important part is actually reading that change before applying it. If someone made an emergency fix by hand during an outage, blindly applying would undo that fix.

So I always check first — if it was a real fix that should stay, I update the code to match it instead of reverting it.

</details>

---

### Q: How do you refactor a Terraform module that's already deployed in production without a destroy/recreate cycle on live resources?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Terraform tracks resources by their name in the code, not some hidden ID, so renaming things can make it think the old one was deleted and a new one created.

I always version my modules, and I test any change in a non-production environment first, checking the plan shows no surprise deletes.

If I genuinely need to rename something without touching the real resource, Terraform has a specific way to handle exactly that, so it updates the tracking instead of recreating anything.

</details>

---

### Q: How do you structure Terraform to safely manage resources across multiple AWS regions from a single codebase?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One provider setting only covers one region, so working across regions needs separate provider blocks, one per region.

What people often miss is that a reusable module doesn't automatically know which region to use — you have to pass that in explicitly.

Skipping that step is exactly how a resource ends up deployed in the wrong region without anyone noticing right away.

</details>

---

### Q: What's the actual access-control model that prevents an engineer from running `terraform apply` against production from their laptop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Just telling people not to do it isn't a real control, it has to be enforced through actual permissions.

The production deploy role can only be used by the CI pipeline itself, no person has a way to use it directly. Write access to the production state is locked down the same way.

I might allow read-only access for debugging, but nobody outside CI can actually apply changes.

</details>

---

### Q: How do you compose outputs from one Terraform stack (e.g., a VPC) as inputs to another (e.g., an ECS cluster) without merging them into one monolithic state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I split infrastructure into layers — network, then compute, then data — so a mistake in one layer can't touch another's state.

To connect them, I read the upstream layer's output as a read-only reference, so the newer layer can use things like the VPC's subnet IDs without ever being able to change the VPC itself.

That connection only ever flows in one direction, which keeps the dependency between stacks clear and predictable.

</details>

---

### Q: A `terraform apply` fails halfway through, applying some resources and erroring on others. What's the actual recovery process?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The good news is Terraform tracks progress as it goes, so anything that succeeded is already recorded correctly in the state.

First I figure out why it actually failed — usually a permissions issue or a service limit — fix that, then run `apply` again. It only touches what's still missing, it doesn't redo what already worked.

If the state itself looks inconsistent afterward, I restore it from the last good backup instead of guessing.

</details>

---

## 8. AWS Infrastructure & Networking

### Q: An EC2 instance is unreachable. What's the deterministic order of checks, and why does that order matter?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check this in the same order every time, so I don't waste time chasing the wrong layer.

Security group first, then the network-level rules, since those need both inbound and outbound allowed, not just one direction like security groups. Then the route table, then the instance itself, and only last, anything on the operating system.

Checking out of order just means chasing symptoms that aren't the actual cause.

</details>

---

### Q: Design a multi-VPC network topology — what's the actual architectural decision between VPC Peering and Transit Gateway, and where does each break down at scale?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Peering connects two networks directly, but it doesn't pass through — if A is connected to B, and B to C, A still can't reach C. That gets hard to manage once you have more than a handful of networks.

Transit Gateway solves this with one central hub that everything connects to, though it costs more and becomes one more thing to watch.

In my experience, I use Peering for a small, stable set of networks, and move to Transit Gateway once there are enough networks that managing individual connections becomes a real burden.

</details>

---

### Q: RDS performance degrades under load. What's the exact diagnostic sequence to isolate whether it's compute, connections, or query-level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

"The database is slow" can mean a few different things, so I check them in order.

First CPU and memory — if that's maxed out, it's a real capacity issue. Then the connection count — if that's climbing, it's usually the app not managing connections properly, not the database itself.

Then I check for slow queries specifically, since a missing index can look exactly like a capacity problem, but the fix there is an index, not a bigger instance.

</details>

---

### Q: What's the actual blast radius when a NAT Gateway fails, and how do you architect against it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A NAT Gateway only covers one availability zone. If every private subnet across all zones points at just one NAT Gateway to save cost, losing that one zone kills outbound internet everywhere, not just there.

The setup I use is one NAT Gateway per zone, so a failure only affects that single zone.

It costs more, but for production, I think it's worth it.

</details>

---

### Q: Walk through what actually changed to cut deployment cost by 40% — name the specific mechanisms, not the category.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

We moved workloads to Kubernetes with autoscaling, so we stopped paying for capacity we didn't actually need around the clock.

We added interruptible instances for workloads that could handle being restarted, which cut compute cost a lot on its own.

We also made our Docker images smaller, cleaned up unused resources like old storage volumes and load balancers nobody was using, and added storage lifecycle rules so old data moved to cheaper storage automatically.

</details>

---

## 9. ECS Fargate + ALB — Production Failure Modes

### Q: A Fargate service behind an ALB shows intermittent 502s for 2-3 minutes during every deployment. Diagnose the full failure chain and the fix.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This usually happens because the new tasks aren't fully ready yet, but the load balancer starts sending them traffic too early.

First, I check how long the app actually takes to start, and then I set the ECS health check grace period to match that, so ECS doesn't mark the task unhealthy while it's still starting up. I also make sure the health check itself is actually checking readiness, not just that the process is running.

For the old tasks, I set the load balancer's deregistration delay so any requests already in progress get time to finish before the task is stopped. In our case, fixing this was about aligning all these timings together, not changing just one setting.

</details>

---

### Q: Precisely distinguish `healthCheckGracePeriodSeconds` from ALB target group `deregistration_delay` — what does each actually control, and what happens if they're confused or misconfigured relative to each other?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

These two settings control opposite ends of a task's life.

The grace period is about new tasks just starting up — it stops ECS from killing them too early. The deregistration delay is about old tasks shutting down — it gives the load balancer time to stop sending them traffic first.

If new tasks keep crashing right when they start, that's the grace period setting. If errors happen specifically when old tasks are being removed, that's the deregistration delay.

</details>

---

### Q: A container's real startup time (90s) exceeds the ALB's health check interval (30s). What's the exact ECS/ALB configuration to prevent 502s here, and why is a shorter interval alone not the fix?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The interval just controls how often the check runs, that part's fine on its own. The real risk is if ECS's grace period is shorter than the actual 90 seconds the app needs to start.

If it is, ECS kills the task before it ever gets a fair chance, no matter how the load balancer's interval is set.

So I'd set the grace period to something like 120 seconds, with real headroom above the measured startup time, and make sure the health check only passes once the app is actually ready.

</details>

---

### Q: Given a 502 in production, how do you determine — from logs alone, without guessing — whether the ALB or the application generated it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A 502 just means the load balancer didn't get a good response from the app, it doesn't tell you why on its own.

I check the load balancer's access logs and look at the specific field showing the app's response code. If that field is empty, the load balancer never even reached the app — that's an infrastructure issue. If it shows a real number, like 500, the app did respond, just with an error — that's a code issue.

That one field tells me exactly where to look next.

</details>

---

## 10. IAM, Secrets & Security Boundaries

### Q: How do you get secrets to a Lambda function at runtime without ever having them touch source control or unencrypted CI logs?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never go directly into config files that live in Git, since even deleting them later doesn't remove them from the history.

They live in a secrets manager instead, and get pulled in and decrypted right at deploy time. The Lambda's own permissions only allow it to access the exact secret it needs, nothing broader.

That way, even if the function were compromised somehow, it couldn't read anything else it doesn't already have access to.

</details>

---

### Q: How does a CI/CD pipeline authenticate into multiple AWS accounts without static, long-lived credentials sitting in the CI system?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Static access keys sitting in CI are a real risk, since they don't expire and are hard to revoke fast if something goes wrong.

What I use instead is a setup where CI requests short-lived credentials tied to a specific role, and that role only trusts that exact pipeline. From there, it assumes a role into the target account with permissions scoped to just that job.

Every one of those actions gets logged too, so there's a full trail if anything ever needs to be checked.

</details>

---

### Q: What's the actual difference in engineering intent between SSM Parameter Store and Secrets Manager — when is using Parameter Store for a secret the wrong call?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Both can encrypt values, so that's not really the difference.

Secrets Manager can automatically rotate credentials on a schedule, Parameter Store can't do that on its own.

So I use Parameter Store for config and secrets that never need to rotate, and Secrets Manager for anything like a database password that genuinely should change regularly.

</details>

---

### Q: How do you structurally guarantee a feature-branch pipeline run can never deploy to production, rather than relying on pipeline YAML conditionals alone?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A branch check written into the pipeline file alone isn't a real wall, since anyone with edit access could accidentally break it.

So I enforce the actual rule at the permissions level — the production role only trusts requests coming from the main branch specifically. Even if the pipeline file gets misconfigured, a feature branch trying to use that role just gets rejected outright.

That's the boundary that actually holds, regardless of what the pipeline file says.

</details>

---

## 11. Serverless — Lambda Architecture

### Q: What's the actual mechanism behind a Lambda cold start, and which architectural levers reduce it versus which just mask it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A cold start is the time it takes to spin up a brand new environment — downloading the code, starting the runtime, and running anything outside the main handler, like setting up a database connection.

That setup only happens once per environment, not on every call, so a common fix is moving that setup code outside the handler function.

For predictable, time-sensitive traffic, keeping a set number of environments warm ahead of time helps, but it doesn't help with a sudden, unplanned spike bigger than what was prepared for.

</details>

---

### Q: Design a Lambda-based async processing pattern using SNS — what failure modes does this introduce that a synchronous call doesn't have, and how do you close them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

SNS can deliver the same message more than once, that's just how it works.

So the function receiving it has to handle a duplicate safely, or you end up with duplicate actions, like two database rows instead of one. I always check a message ID against a small lookup table before processing it, so a repeat message is just ignored.

I also add a backup queue for anything that keeps failing, so it doesn't just quietly disappear.

</details>

---

### Q: Design a single-codebase Serverless Framework deployment pipeline that deploys the same application across dev/stg/prod AWS accounts with clean environment-specific configuration and no secret leakage between environments.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One codebase, with a separate config file per environment for anything that isn't sensitive, like a network ID.

Secrets are never in those files — they get pulled from the secrets manager at deploy time, scoped so each account can only read its own.

The pipeline picks both the AWS account and the matching config together, so they can never drift apart, and the actual deployment package gets built once and promoted through every environment as-is.

</details>

---

## 12. Observability — Monitoring, Logging & Auditing

### Q: Design the logging/auditing layer for an AWS account — what does each service actually give you, and what's the gap if you only use one?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CloudTrail tells you who changed what. Flow Logs tell you what network traffic actually happened. Config tells you what a resource's setup looked like over time.

None of them alone gives the full picture — you need all three together, lined up by time, to really understand an incident.

I ship all of it into one central place so I can search across everything at once during an investigation, rather than checking three separate consoles under time pressure.

</details>

---

### Q: A Redis cluster shows frequent evictions. What's the actual diagnostic path to determine whether this is a capacity problem or an application-design problem?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Evictions mean memory is full, but the reason matters.

If memory use keeps climbing with no leveling off, that usually points to an app bug — keys being written without an expiry, so they build up forever. If usage is genuinely large but steady, that's real undersizing, and I'd scale the cluster.

I also always check the eviction setting itself is actually right for a cache, not something that just stops accepting writes once it's full.

</details>

---

### Q: What does a production-grade observability stack actually need to cover, beyond "we have CloudWatch"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Basic metrics and logs alone can't show you a slow request as it moves across five different services — for that, real tracing is needed so you can see the full path of one request.

I also alert on things users actually feel, like error rate and slow response times, not just CPU usage, since CPU alone can miss a real problem.

And backups only really count if the restore has been tested, not just set up and left alone.

</details>

---

### Q: Define an incident response process that survives beyond "we look at the dashboard and fix it" — what's the actual structured loop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The loop is simple and repeats the same way every time. Detect it fast using real symptoms like error rate, dig in using logs and traces to find the actual cause, then fix the immediate problem first, even before the full root cause is understood.

After that, I always run a blameless review with real action items, not just a report nobody follows up on.

Without that last step, the same problem just comes back later.

</details>

---

## 13. Linux / Scripting — Systems Operations

### Q: Write the exact command to purge log files older than 30 days and larger than 50MB, and explain the failure mode of getting the `find` predicate order wrong.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is:

```bash
find /var/log -type f -size +50M -mtime +30 -delete
```

The key part is `-type f` — without it, folders could match too, and combined with delete, that could remove whole directories by accident.

I always swap the delete for a print first and check the list of what would actually get removed, before running the real command, especially on a server I haven't worked on before.

</details>

---

### Q: Write a script that counts ERROR occurrences in a log file, and explain why a naive substring match is a correctness risk at production log volume.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A simple search for the word "ERROR" anywhere in a line will also match something like "error_count field updated," which isn't a real error at all.

That quietly gives a wrong number with no warning that anything's off. I always match on a proper word boundary, or better, check the actual log level field if the logs are structured.

At real scale, this kind of counting belongs in a log search tool like CloudWatch Insights, not a small local script.

</details>

---

### Q: Automate Nginx installation across a fleet of 10 servers with Ansible — what's the idempotency guarantee the `apt` module gives you that a raw shell command doesn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A raw shell command just runs every single time, no matter what.

Ansible's own install module checks the current state first, and only makes a change if it's actually needed, so running it again just reports nothing changed instead of doing the work over.

I always use these built-in modules instead of raw shell commands, since that gives me an honest signal when something unexpected actually changed.

</details>

---

### Q: What's the actual purpose of separating `roles/`, `group_vars/`, and `host_vars/` in an Ansible project, beyond directory tidiness?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This isn't just about tidiness, it's about control.

Roles hold reusable automation that shouldn't change between projects. `group_vars` sets defaults for a whole group of servers. `host_vars` overrides just one specific server, and that always wins over the group setting.

This setup is exactly what lets the same automation run safely across dev, staging, and prod, without changing the actual logic each time.

</details>

---

## 14. System Design — End-to-End Architecture

### Q: Describe a production multi-account AWS architecture end-to-end — the actual isolation boundaries, the deployment path, and where each control lives.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In my experience, the whole point of using multiple AWS accounts is keeping failures contained — a mistake in dev should never be able to touch prod.

We run separate accounts per environment, Lambda for event-driven work and containers behind a load balancer for everything else, a mix of relational and key-value databases depending on the need, and queues to keep services loosely connected instead of calling each other directly.

Terraform is modular with its own state per account, and deploys only happen through CI using short-lived access, never from anyone's own laptop. Production specifically needs manual approval, enforced through real permissions, not just a pipeline setting, and secrets always come from a secrets manager, never from the code itself.

</details>

---

<div align="center">

⭐ *If this helped you prep, consider starring the repo.*

</div>
