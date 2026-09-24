<div align="center">

# DevOps Interview Q&A — Senior / Principal Level

![Topics](https://img.shields.io/badge/topics-14-blue)
![Level](https://img.shields.io/badge/level-Senior--Principal-orange)
![Format](https://img.shields.io/badge/format-Markdown-informational)
![Stack](https://img.shields.io/badge/stack-AWS%20%7C%20K8s%20%7C%20Terraform%20%7C%20CI%2FCD-success)

Simple, plain-English candidate answers — short sentences, the way you'd actually say it out loud in an interview.

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

Each topic file uses plain-English, first-person candidate answers in `<details>` flashcards, same style as this README.

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

There are three types. A **bind mount** connects a folder on the host straight into the container. It's fast, but the container can now see real host files, and file ownership can get messy. A **named volume** is fully managed by Docker — it's the safe choice, and it doesn't get deleted by normal cleanup commands. An **anonymous volume** is like a named one, but with no name, so it's easy to forget about, and it can quietly fill up disk space over time. I always use named volumes for anything like a database.

</details>

---

### Q: `CMD` vs `ENTRYPOINT` — what's the actual execution model, and when does mixing them break a container's signal handling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`ENTRYPOINT` is the main command that runs. `CMD` just gives it default arguments. One thing to watch for — if you write `CMD npm start` as plain text instead of the list format, Docker wraps it in a shell, and that shell becomes the main process instead of your app. So when Docker tries to stop the container the normal way, the app never gets the signal, and it just hangs until it's force-killed. I always use the list format, like `ENTRYPOINT ["node", "server.js"]`, so shutdown works properly.

</details>

---

### Q: A container restarts and all data is gone. What's actually happening at the storage-driver level, and how do you architect around it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This isn't a bug, it's expected. Every container has its own storage space, and that space gets wiped when the container is removed. If the app wrote data to a path with no volume attached, that data was never going to survive a restart. The fix is simple — mount a real volume, or a `PersistentVolumeClaim` in Kubernetes, at every path the app actually needs to keep.

</details>

---

### Q: How do you reclaim disk on a Docker host without risking an active build cache or in-use image?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never run a full cleanup command blindly, especially on a CI server. It clears out every image that isn't currently in use, and that can delete a build cache another job still needs, which just slows everything down. Instead, I clean up in a targeted way — only images older than a certain age, for example. I also track disk usage as a real metric, so I catch a problem early, instead of finding out when a build suddenly fails.

</details>

---

## 2. Kubernetes — Orchestration & Control Plane

### Q: What do taints and tolerations actually enforce at the scheduler level, and what do they *not* protect against?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A **taint** on a node blocks pods from being placed there. A **toleration** on a pod just lets it get past that block — it doesn't force the pod to go there. This only controls scheduling, it's not real security. A pod placed some other way, or one that's already running, isn't stopped by a taint at all. If I actually need dedicated nodes, I pair the taint with **node affinity** too. If I need real isolation, I add a `NetworkPolicy` on top of that.

</details>

---

### Q: Is pod-to-pod traffic allowed by default, and how do you enforce a default-deny posture without breaking DNS or control-plane traffic?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Yes, by default every pod can talk to every other pod. There's no restriction out of the box. The common mistake when people turn on a deny-all rule for the first time is it breaks DNS, since the DNS service lives in a different namespace and needs its own explicit rule to still be allowed. I always test this in a non-production namespace first, and I add a clear rule allowing DNS traffic before rolling it out anywhere else.

</details>

---

### Q: `StatefulSet` vs `Deployment` — what specific guarantees does a `StatefulSet` provide that make it non-optional for stateful workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A `Deployment` gives pods random names with no set order, which is fine for apps that don't hold state. A `StatefulSet` gives pods fixed names, its own storage per pod, and starts them one at a time in order — that's what something like a database actually needs. Running a database on a plain `Deployment` looks fine at first, but breaks the moment you do a rolling update. That's exactly why `StatefulSet` exists.

</details>

---

### Q: A pod is stuck in `CrashLoopBackOff`. Walk through the exact diagnostic sequence and what each signal tells you.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`CrashLoopBackOff` just means the pod keeps failing, and Kubernetes keeps trying again with a longer wait each time. It doesn't tell you why. I always start by checking the pod's exit code and events, then I check the logs from the last crash, since the current instance already restarted with a clean slate. If the exit code is `137`, that almost always means it ran out of memory, so I'd check the memory limit against real usage. Anything else usually means the app itself hit an error.

</details>

---

### Q: An application deployed to EKS isn't reachable externally. What's the exact layer-by-layer isolation boundary you check, in order?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check this in order, because guessing wastes time. First, I make sure the Service is actually pointing at a healthy pod. Then I check the port setting, then the load balancer setup, then the security groups between the load balancer and the nodes, and finally DNS. In my experience, the security group step is where most of these problems actually turn out to be.

</details>

---

### Q: How does cross-pod communication actually traverse the stack inside an EKS cluster, mechanically?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In EKS, every pod gets a real IP address straight from the VPC, it's not some fake separate network. That's actually why the number of pods per node is limited — it depends on how many IP addresses that instance type can hand out. When one pod talks to a service by name, it goes through the cluster's DNS first to find the address, and then gets routed to the right pod behind the scenes. For big clusters, I'd switch that routing to a mode that scales better than the older default.

</details>

---

## 3. CI/CD — Jenkins & Pipeline Engineering

### Q: What are the trade-offs between Jenkins deployment models (static EC2, Docker, Helm-on-EKS), and which failure modes does each eliminate or introduce?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A plain server install is simple, but it's a single point of failure, and how many builds can run at once is stuck at whatever that one server can handle. Running it in Docker helps a little, but builds can still leave old files behind for the next job. What I actually run is Jenkins on Kubernetes — every single build gets its own fresh, throwaway worker that's deleted right after, so nothing carries over between builds, and capacity grows with the cluster automatically.

</details>

---

### Q: A Jenkins pipeline fails intermittently, not on every run. What's the deterministic debugging sequence, and how do you distinguish a flaky test from an infrastructure race condition?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't call something "flaky" until I've actually checked. I first see if the failures line up with new workers starting up — if they do, that's a timing issue with the infrastructure, not a bad test. A good trick is running the job again with everything set to run one at a time. If the failure disappears, it was a resource conflict, not the test itself. Only after ruling all that out do I actually mark a test as flaky and track it properly.

</details>

---

### Q: How do you manage Jenkins plugin upgrades on a Helm-deployed instance without an untested plugin breaking the production pipeline?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never update plugins by clicking around in the Jenkins interface — on a Kubernetes setup, that gets wiped out on the next deploy anyway. Plugin versions are set in one config file, and that's the real source of truth. Before I bump a version, I test it on a separate, non-production Jenkins first, using our real pipelines. Then I roll it out in a way that automatically rolls back if something fails its health check.

</details>

---

### Q: The Jenkins admin credential is lost and the controller runs on Kubernetes with no external secret backup. What's the recovery path, and how do you prevent this from being a single point of failure again?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If this is still the first-time setup, the starting password is sitting in a file I can read directly. If it's a real lost account after that, I have to get inside the server and reset it through a script, which is not something you want to be doing during an actual incident. Going forward, the real fix is connecting Jenkins to the company's normal login system, so there's no single admin password to lose in the first place. I also make sure the Jenkins data gets backed up on a schedule, just in case.

</details>

---

### Q: Design a rollback strategy set that covers every layer of a deployment — application, infrastructure, and pipeline state.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Rollback isn't one single thing, every layer needs its own plan. For the app in Kubernetes, there's a direct command to undo the last change, instantly, no rebuild needed. For Terraform, there's no real rollback command, so reviewing the plan carefully before applying is the real safety net. For source code, I always use a safe revert, never a hard reset on a shared branch, since that rewrites history everyone else already has.

</details>

---

## 4. Git — Version Control Internals

### Q: Explain the branching model you enforce, and specifically why `merge` vs `rebase` changes the risk profile of a shared branch.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**Merge** just adds a new commit and keeps history as it really happened, so it's always safe on a shared branch. **Rebase** rewrites commit history with brand new IDs, and doing that on a branch others already have breaks their copy too. So my rule is simple — I only rebase my own branch before pushing it, never after it's shared with the team. Day to day, I prefer short branches that get merged into `main` quickly.

</details>

---

### Q: How do you cleanly squash a range of commits before merging, and what breaks if you do it after pushing to a shared branch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use an interactive rebase to squash a few messy commits into one clean commit. The problem is, if that branch is already shared, this creates new commit IDs and breaks everyone else's copy of it. So I always use a safer force-push option that protects against overwriting someone else's work by accident. And if it's genuinely already shared, I give people a heads-up before I touch it.

</details>

---

### Q: The `.git` directory is gone from a working copy. What's actually recoverable, and what isn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The `.git` folder holds the entire history of the project. Losing it means losing everything that was never pushed anywhere else. If there's a remote and everything was already pushed, it's easy — just get a fresh copy, nothing's actually lost. But any local commit that was never pushed is gone for good. That's exactly why I push often, and I don't let real work sit only on my own machine for too long.

</details>

---

### Q: `git fetch` vs `git pull` — what's the actual difference in terms of working-tree risk?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git fetch` just downloads the latest changes, it doesn't touch your own files at all, so it's completely safe. `git pull` does that same download, plus it automatically merges it into your current work — and if you have local changes that conflict, that can leave a mess. I use `pull` for everyday work, but I never let it run inside a script or automation without a safety setting that stops it from silently merging something unexpected.

</details>

---

## 5. Terraform — State & Fundamentals

### Q: How do you bring an out-of-band-created AWS resource under Terraform management without recreating it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is `terraform import`, but it only updates the state file — it does not write the matching code for you. If the code doesn't already match the real resource, the next plan will try to change or even delete it. So my real steps are: write the matching code first, run the import, then run `plan` and make sure it shows no changes before I trust it.

</details>

---

### Q: What specifically breaks when Terraform state is kept local in a multi-engineer team, beyond "it's not backed up"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The real problem is there's no locking. If two people run `apply` at the same time, one silently overwrites the other's changes, and now Terraform doesn't even know some real resources exist anymore. A shared backend with locking isn't optional for a team, it's required from day one. I also turn on versioning on that storage, since that's the real way to undo a bad state file.

</details>

---

### Q: What does a real Terraform testing/validation pipeline enforce before `apply` ever runs against production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform validate` only checks the syntax, it has no idea if something's actually valid on AWS's side. So I layer several checks — a format check, then validate, then a proper linter for real provider-level mistakes, then a security scan, and then a plan that a real person reviews on the pull request. Production `apply` is always behind manual approval. The whole idea is catching mistakes early, not at apply time.

</details>

---

## 6. Terraform — Multi-Account Architecture

### Q: Design the module/state topology for a Terraform codebase spanning dev/staging/prod across separate AWS accounts.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build shared, reusable pieces of code for things like a VPC or a database, with no environment-specific logic baked in. Then each environment gets its own small folder that calls those shared pieces with its own settings. Each environment also gets its own separate state file, tied to its own account, so nothing overlaps. CI switches into the right account before it ever touches anything real.

</details>

---

### Q: How do you prevent a Terraform state read/write in one AWS account from ever touching another account's state, structurally rather than by convention?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't rely on people remembering to point at the right config — that always breaks eventually. Instead, each account gets its own storage bucket for state, and the access policy only allows that account's own pipeline to touch it. So even a mistake in a pipeline setting physically can't read or write another account's state. That's a real wall, not just a rule written down somewhere.

</details>

---

### Q: How do you keep secrets (DB credentials, API keys) out of both the Terraform codebase and the state file, given that Terraform state stores resource attributes in plaintext by default?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The thing most people miss is that Terraform's state file is plain text by default, so even a secret passed in as a variable ends up sitting right there in it. My rule is simple — secrets never go in as raw variables. They live in a real secrets manager, and Terraform just references them. On top of that, I turn on encryption on the state storage, and I lock down who's even allowed to read it.

</details>

---

### Q: Trace exactly what happens end-to-end when a Terraform change merges to `main` in a CI/CD-driven multi-account pipeline.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, Terraform connects to the right backend for that environment. Then it builds a plan, and that plan gets posted somewhere a real person can review it — that's the actual safety check. After manual approval for production specifically, the pipeline switches into the right account and applies that exact same plan, not a new one. Every log and plan gets saved, tied back to the commit that started it.

</details>

---

## 7. Terraform — Drift, Recovery & Advanced Ops

### Q: Someone manually changes a resource in the AWS console that Terraform manages. What does `terraform plan` actually show you, and how do you resolve it correctly versus incorrectly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform plan` shows a change that would undo the console edit and put things back to what's in the code — that's totally normal, Terraform has no idea who made a change, it just compares. The important part is actually reading that change before applying it. If someone made an emergency fix by hand during an outage, blindly applying would undo that fix. So I check first — if it was a real, needed fix, I update the code to match it instead of reverting it.

</details>

---

### Q: How do you refactor a Terraform module that's already deployed in production without a destroy/recreate cycle on live resources?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Terraform tracks resources by their name in the code, not some hidden ID, so renaming things can make it think the old one was deleted and a new one created. I always version modules, and I test any change in a non-production environment first, checking that the plan shows no surprise deletes. If I genuinely need to rename something without actually touching the real resource, there's a specific block made for exactly that.

</details>

---

### Q: How do you structure Terraform to safely manage resources across multiple AWS regions from a single codebase?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One provider setting only covers one region, so working across regions needs separate, named provider blocks — one per region. The thing people miss is that reusable modules don't automatically know which region to use, you have to pass that in yourself. Skipping that step is exactly how a resource ends up quietly deployed in the wrong region.

</details>

---

### Q: What's the actual access-control model that prevents an engineer from running `terraform apply` against production from their laptop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Just telling people not to do it isn't a real control, it has to be enforced through actual permissions. The production deploy role can only be used by the CI pipeline itself — no person has a way to use it directly, even if they wanted to. Write access to the production state is locked down the same way. I might allow read-only access for debugging, but nobody outside CI can ever actually apply.

</details>

---

### Q: How do you compose outputs from one Terraform stack (e.g., a VPC) as inputs to another (e.g., an ECS cluster) without merging them into one monolithic state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I split infrastructure into layers — network, then compute, then data — so a mistake in one layer can't touch another's state. To connect them, I read the upstream layer's output as a read-only reference, so the newer layer can use things like the VPC's subnet IDs without ever being able to change the VPC itself. The connection only ever flows in one direction, never both ways.

</details>

---

### Q: A `terraform apply` fails halfway through, applying some resources and erroring on others. What's the actual recovery process?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The good news is this is fully recoverable. Terraform saves progress as it goes, so anything that succeeded is already tracked correctly in the state. I first figure out why it actually failed — usually a permissions or a limit issue — fix that, then just run `apply` again, and it only touches what's still left to do. If the state itself ever looks wrong, I restore an earlier saved version instead of guessing.

</details>

---

## 8. AWS Infrastructure & Networking

### Q: An EC2 instance is unreachable. What's the deterministic order of checks, and why does that order matter?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check this in the same order every time, so I don't waste time. Security group first, then the network-level rules, since those need both directions allowed, not just one like security groups. Then the route table, then the instance itself, and only last, anything on the operating system, like a local firewall. Checking out of order just means chasing symptoms that aren't the real cause.

</details>

---

### Q: Design a multi-VPC network topology — what's the actual architectural decision between VPC Peering and Transit Gateway, and where does each break down at scale?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**VPC Peering** connects two networks directly, but it doesn't pass through — if A is connected to B, and B is connected to C, A still can't reach C. That gets hard to manage once you have more than a handful of networks. **Transit Gateway** fixes this with one central hub that everything connects to, but it costs more and becomes one bigger thing to keep an eye on. I use Peering for a small, stable set of networks, and move to Transit Gateway once I actually need that hub-style setup.

</details>

---

### Q: RDS performance degrades under load. What's the exact diagnostic sequence to isolate whether it's compute, connections, or query-level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

"The database is slow" can mean a few different things, so I check them in order. First CPU and memory — if that's maxed out, it's a real capacity problem. Then the number of connections — if that's climbing, it's almost always the app not managing connections properly, not the database itself. Then I check for slow queries — a missing index can look exactly like a capacity problem, but the fix there is an index, not a bigger server.

</details>

---

### Q: What's the actual blast radius when a NAT Gateway fails, and how do you architect against it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A NAT Gateway only covers one zone. If every private subnet across all zones points at just one NAT Gateway to save money, losing that one zone kills outbound internet access everywhere, not just in that zone. The right setup is one NAT Gateway per zone, so a failure only affects that single zone. It costs more, but it's worth it for production.

</details>

---

### Q: Walk through what actually changed to cut deployment cost by 40% — name the specific mechanisms, not the category.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

We moved workloads to Kubernetes with autoscaling, so we stopped paying for capacity we didn't actually need all day. We added cheaper, interruptible instances for workloads that could handle being restarted, which cut compute cost a lot on its own. We also made our Docker images smaller, cleaned up unused resources like old storage volumes and load balancers nobody was using, and added storage lifecycle rules to move old data to cheaper storage automatically.

</details>

---

## 9. ECS Fargate + ALB — Production Failure Modes

### Q: A Fargate service behind an ALB shows intermittent 502s for 2-3 minutes during every deployment. Diagnose the full failure chain and the fix.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This comes down to three timers that don't line up. There's how long the app actually takes to start, how long ECS waits before it trusts a new task, and how long the load balancer waits before sending it real traffic. If ECS's wait time is shorter than the app's real startup time, it kills the task before it's even ready — and that looks exactly like a 502 to the user. We fixed it by giving ECS enough wait time for the real startup, using a health check that only passes once the app is actually ready, and giving old tasks enough time to finish their current requests before they get shut down.

</details>

---

### Q: Precisely distinguish `healthCheckGracePeriodSeconds` from ALB target group `deregistration_delay` — what does each actually control, and what happens if they're confused or misconfigured relative to each other?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`healthCheckGracePeriodSeconds` is about new tasks just starting up — it stops ECS from killing them too soon. `deregistration_delay` is about old tasks shutting down — it gives the load balancer time to stop sending them traffic first. If new tasks keep crashing right after they start, that's the grace period setting. If errors happen specifically when old tasks are being removed, that's the deregistration delay. They fix two completely different moments in a deploy, so mixing them up means tuning the wrong setting for the actual problem.

</details>

---

### Q: A container's real startup time (90s) exceeds the ALB's health check interval (30s). What's the exact ECS/ALB configuration to prevent 502s here, and why is a shorter interval alone not the fix?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The check interval just controls how often the health check runs, that part's fine on its own. The real problem is if ECS's grace period is shorter than the actual 90 seconds the app needs — if it is, ECS kills the task before it ever gets a fair chance to become healthy. So I set that grace period to something like 120 seconds, with real headroom, and I make sure the health check only passes once the app is truly ready, not just technically running.

</details>

---

### Q: Given a 502 in production, how do you determine — from logs alone, without guessing — whether the ALB or the application generated it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A 502 just means the load balancer didn't get a good response from the app, it doesn't tell you why on its own. I go straight to the load balancer's access logs and check the specific field showing the app's response code. If that field is empty, the load balancer never even reached the app — that's an infrastructure problem. If it shows a real number, like a `500`, the app did respond, just with an error — that's a code problem. That one field tells me exactly where to look next.

</details>

---

## 10. IAM, Secrets & Security Boundaries

### Q: How do you get secrets to a Lambda function at runtime without ever having them touch source control or unencrypted CI logs?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never go directly into config files that live in Git, since even deleting them later doesn't remove them from the history. They live in a secrets manager instead, and get pulled in and decrypted right at deploy time. The Lambda's own permissions only allow access to the exact secret it needs, nothing broader. That way, even if the function were somehow compromised, it couldn't read anything else.

</details>

---

### Q: How does a CI/CD pipeline authenticate into multiple AWS accounts without static, long-lived credentials sitting in the CI system?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Access keys that sit in CI forever are a real risk, they don't expire, and they're hard to revoke fast if something goes wrong. Instead, I use a system where CI requests short-lived credentials tied to a specific role, and that role only trusts that exact pipeline. From there, it switches into each target account with permissions scoped tightly to just that job. Every one of those actions gets logged, so there's a full trail if anything ever goes wrong.

</details>

---

### Q: What's the actual difference in engineering intent between SSM Parameter Store and Secrets Manager — when is using Parameter Store for a secret the wrong call?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Both can encrypt values, so that's not really the difference. Secrets Manager can automatically rotate credentials on a schedule, Parameter Store can't do that on its own. So I use Parameter Store for config and secrets that never need rotating, and Secrets Manager for anything like a database password that genuinely should change regularly.

</details>

---

### Q: How do you structurally guarantee a feature-branch pipeline run can never deploy to production, rather than relying on pipeline YAML conditionals alone?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A branch check written into the pipeline file alone isn't a real wall, since anyone with edit access could accidentally break it. So I enforce the real rule at the permissions level — the production role only trusts requests coming from the `main` branch specifically. Even if the pipeline file gets messed up, a feature branch trying to use that role just gets rejected outright. That's the boundary that actually holds, no matter what the pipeline file says.

</details>

---

## 11. Serverless — Lambda Architecture

### Q: What's the actual mechanism behind a Lambda cold start, and which architectural levers reduce it versus which just mask it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A cold start is the time it takes to spin up a brand new environment — download the code, start the runtime, and run anything outside the main handler, like setting up a database connection. That setup only happens once per environment, not on every single call, so the fix is moving that code outside the handler. For traffic that's predictable and time-sensitive, keeping a set number of environments warm ahead of time helps — but it doesn't help with a sudden, unplanned spike bigger than what was prepared for.

</details>

---

### Q: Design a Lambda-based async processing pattern using SNS — what failure modes does this introduce that a synchronous call doesn't have, and how do you close them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

SNS can deliver the same message more than once, that's just how it works. So the function receiving it has to handle a duplicate safely, or you end up with duplicate actions, like two database rows instead of one. I always check a message ID against a small lookup table before actually processing it, so a repeat message is just ignored. I also add a backup queue for anything that keeps failing, so it doesn't just quietly disappear.

</details>

---

### Q: Design a single-codebase Serverless Framework deployment pipeline that deploys the same application across dev/stg/prod AWS accounts with clean environment-specific configuration and no secret leakage between environments.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One codebase, with a separate config file per environment for anything that isn't sensitive, like a network ID. Secrets are never in those files at all — they get pulled from the secrets manager at deploy time, scoped so each account can only read its own. The pipeline picks both the AWS account and the matching config together, so they can never drift apart from each other. And the actual deployment package gets built once, then promoted through every environment as-is.

</details>

---

## 12. Observability — Monitoring, Logging & Auditing

### Q: Design the logging/auditing layer for an AWS account — what does each service actually give you, and what's the gap if you only use one?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CloudTrail tells you who changed what. Flow Logs tell you what network traffic actually happened. AWS Config tells you what a resource's setup looked like over time. None of them alone gives the full picture — you need all three together, lined up by time, to really understand an incident. I ship all of it into one central place, so I can search across everything at once during an investigation.

</details>

---

### Q: A Redis cluster shows frequent evictions. What's the actual diagnostic path to determine whether this is a capacity problem or an application-design problem?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Evictions mean memory is full, but the actual reason matters. I check if memory use keeps climbing with no leveling off — that usually means the app is writing data without ever setting it to expire, which is an app bug, not a sizing problem. If usage is genuinely large but steady, that's real undersizing, and I'd scale the cluster. I also always double-check the eviction setting is actually right for a cache, not something that just stops accepting writes when full.

</details>

---

### Q: What does a production-grade observability stack actually need to cover, beyond "we have CloudWatch"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Basic metrics and logs alone can't show you a slow request as it travels across five different services — for that you need real tracing, so you can see the full path of one request. I also alert on things users actually feel, like error rate and slow response times, not just server CPU, since CPU alone can miss real problems. And backups only actually count if the restore has been tested, not just set up and forgotten.

</details>

---

### Q: Define an incident response process that survives beyond "we look at the dashboard and fix it" — what's the actual structured loop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The loop is simple — detect it fast using real symptoms like error rate, dig in using logs and traces to find the actual cause, then fix the immediate problem first, even before the deeper root cause is fully understood. After that, I always run a blameless review with real action items, not just a report nobody follows up on. Without that last step, the same problem just comes back later.

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

The key part is `-type f` — without it, folders could match too, and combined with delete, that could wipe out whole directories by accident. I always swap the delete part for a print first, and check the list of what would actually get removed, before running the real command, especially on a server I haven't worked on before.

</details>

---

### Q: Write a script that counts ERROR occurrences in a log file, and explain why a naive substring match is a correctness risk at production log volume.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A simple search for the word "ERROR" anywhere in a line will also match something like "error_count field updated," which isn't a real error at all. That quietly gives a wrong number, with no warning that anything's off. I always match on a proper word boundary, or better, check the actual log level field if the logs are structured. At real scale, this kind of counting belongs in a proper log search tool, not a small local script.

</details>

---

### Q: Automate Nginx installation across a fleet of 10 servers with Ansible — what's the idempotency guarantee the `apt` module gives you that a raw shell command doesn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A raw shell command just runs every single time, no matter what. Ansible's own install module checks the current state first, and only makes a change if it's actually needed, so running it again reports nothing changed instead of doing the work over. I always use these built-in modules instead of raw shell commands, since that gives me a real, honest signal when something unexpected actually changed.

</details>

---

### Q: What's the actual purpose of separating `roles/`, `group_vars/`, and `host_vars/` in an Ansible project, beyond directory tidiness?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This isn't just about keeping things tidy, it's about control. **Roles** hold reusable automation that shouldn't change between projects. **`group_vars`** set defaults for a whole group of servers. **`host_vars`** override just one specific server, and that always wins over the group setting. This setup is exactly what lets the same automation run safely across dev, staging, and prod, without changing the actual logic each time.

</details>

---

## 14. System Design — End-to-End Architecture

### Q: Describe a production multi-account AWS architecture end-to-end — the actual isolation boundaries, the deployment path, and where each control lives.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The whole point of using multiple AWS accounts is keeping failures contained — a mistake in dev should never be able to touch prod. I run separate accounts per environment, Lambda for event-driven work and containers behind a load balancer for everything else, a mix of relational and key-value databases depending on the need, and queues to keep services loosely connected instead of calling each other directly. Terraform is modular, with its own state per account, and deploys only happen through CI using short-lived access, never from anyone's own laptop. Production specifically needs manual approval, enforced through real permissions, not just a setting in the pipeline file, and secrets always come from a secrets manager, never from the code itself.

</details>

---

<div align="center">

⭐ *If this helped you prep, consider starring the repo.*

</div>
