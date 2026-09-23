<div align="center">

# DevOps Interview Q&A — Senior / Principal Level

![Topics](https://img.shields.io/badge/topics-14-blue)
![Level](https://img.shields.io/badge/level-Senior--Principal-orange)
![Format](https://img.shields.io/badge/format-Markdown-informational)
![Stack](https://img.shields.io/badge/stack-AWS%20%7C%20K8s%20%7C%20Terraform%20%7C%20CI%2FCD-success)

Quick, plain-English candidate answers — short and to the point, the way you'd actually say it out loud in an interview.

</div>

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

There are three types. A **bind mount** points straight at a folder on the host, so it's fast, but the container can now see host files, and ownership is just raw user IDs, which causes a lot of permission errors. A **named volume** is fully managed by Docker, it's safer, and it survives cleanup commands. An **anonymous volume** is like a named one but has no name, so it's easy to forget and leave behind, filling up disk space over time. In my experience, I always use named volumes for anything like a database.

</details>

---

### Q: `CMD` vs `ENTRYPOINT` — what's the actual execution model, and when does mixing them break a container's signal handling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**`ENTRYPOINT`** is the main process that runs, and `CMD` just gives it default arguments. The big catch here is if you write `CMD npm start` instead of the array form, Docker wraps it in a shell, and that shell becomes the main process instead of your app. So when Docker tries to stop the container nicely, the signal never reaches your app, and it just hangs until it gets killed. We fixed this by always using the array form, like `ENTRYPOINT ["node", "server.js"]`, so shutdown actually works.

</details>

---

### Q: A container restarts and all data is gone. What's actually happening at the storage-driver level, and how do you architect around it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This isn't a bug, it's expected. Every container has its own writable space, and when the container gets removed, that space is wiped too. If your app wrote data to a path with no volume attached, it was never going to survive a restart. In my experience, the fix is simple — mount a real volume, or a `PersistentVolumeClaim` in Kubernetes, at every path your app actually needs to keep.

</details>

---

### Q: How do you reclaim disk on a Docker host without risking an active build cache or in-use image?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never run `docker system prune -a` blindly, especially on a CI server. It clears out every image not in use right now, which can wipe out a build cache another job still needs, and that just slows everything down. Instead, I scope it — something like `docker image prune -f --filter "until=72h"` — so I only clean up genuinely old stuff. We also track disk usage as a real metric, so we catch it early instead of during a failed build.

</details>

---

## 2. Kubernetes — Orchestration & Control Plane

### Q: What do taints and tolerations actually enforce at the scheduler level, and what do they *not* protect against?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A **taint** on a node blocks pods from landing there, and a **toleration** on a pod just lets it get past that block — it doesn't force the pod to go there. The big catch is this only controls scheduling, it's not real security. A pod placed some other way, or already running, isn't stopped by a taint. In practice, if I actually need dedicated nodes, I pair the taint with **node affinity**, and if I need real isolation, I add a `NetworkPolicy` on top.

</details>

---

### Q: Is pod-to-pod traffic allowed by default, and how do you enforce a default-deny posture without breaking DNS or control-plane traffic?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Yes, by default every pod can talk to every other pod, there's no restriction out of the box. The big catch when people turn on a deny-all policy for the first time is it breaks DNS, because CoreDNS lives in a different namespace and needs an explicit rule to allow it through. We always test this in a non-prod namespace first, and we add a clear allow rule for DNS traffic before rolling it out anywhere else.

</details>

---

### Q: `StatefulSet` vs `Deployment` — what specific guarantees does a `StatefulSet` provide that make it non-optional for stateful workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A `Deployment` gives pods random names with no order, which is fine for stateless apps. A `StatefulSet` gives stable names, stable storage, and starts pods one at a time in order, which is what something like a database or Kafka actually needs. In my experience, running stateful software on a plain `Deployment` looks fine at first, but breaks the moment you do a rolling update. That's exactly why `StatefulSet` exists.

</details>

---

### Q: A pod is stuck in `CrashLoopBackOff`. Walk through the exact diagnostic sequence and what each signal tells you.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`CrashLoopBackOff` just means the pod keeps failing and Kubernetes keeps retrying with a longer wait each time — it doesn't tell you why. I always start with `kubectl describe pod` to see the exit code, then `kubectl logs --previous` to see what the app actually printed before it died. If the exit code is `137`, that's almost always **out of memory**, and I'd check the memory limit against real usage. If it's something else, it's usually just a bug in the app itself.

</details>

---

### Q: An application deployed to EKS isn't reachable externally. What's the exact layer-by-layer isolation boundary you check, in order?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check this in order, because guessing wastes time. First, `kubectl get endpoints` to make sure the service is actually pointing at a healthy pod. Then the service port, then the ingress or load balancer setup, then security groups between the load balancer and the nodes, and finally DNS. In my experience, the security group step is where most EKS traffic actually breaks.

</details>

---

### Q: How does cross-pod communication actually traverse the stack inside an EKS cluster, mechanically?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In EKS, every pod gets a real IP address straight from the VPC, it's not some fake overlay network. That's actually why pod count per node is limited — it's tied to how many IPs that instance type can hand out. Service traffic goes through CoreDNS for the name lookup, and then `kube-proxy` routes it to the right pod under the hood. For big clusters, I switch `kube-proxy` to **IPVS mode**, since it scales a lot better than the older method.

</details>

---

## 3. CI/CD — Jenkins & Pipeline Engineering

### Q: What are the trade-offs between Jenkins deployment models (static EC2, Docker, Helm-on-EKS), and which failure modes does each eliminate or introduce?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A plain EC2 install is simple but it's a single point of failure, and build capacity is stuck at whatever that one box can handle. Running the controller in Docker helps a bit, but builds can still leave leftover junk behind for the next job. What I actually run is Jenkins on Kubernetes with Helm — every single build gets its own throwaway agent pod that's deleted right after, so nothing ever bleeds between builds, and capacity scales with the cluster.

</details>

---

### Q: A Jenkins pipeline fails intermittently, not on every run. What's the deterministic debugging sequence, and how do you distinguish a flaky test from an infrastructure race condition?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't call it "flaky" until I've actually proven it. I check if the failures line up with new agents starting up — if they do, that's an infrastructure timing issue, not a bad test. A good trick is rerunning the job with concurrency set to one — if the failure disappears, it was a resource conflict, not the test itself. Only after ruling all that out do I actually mark a test as flaky and put a ticket on it.

</details>

---

### Q: How do you manage Jenkins plugin upgrades on a Helm-deployed instance without an untested plugin breaking the production pipeline?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never update plugins by clicking around in the Jenkins UI — on Helm, that gets wiped out on the next deploy anyway. Plugin versions are pinned in `values.yaml`, that's the single source of truth. Before bumping a version, I test it on a separate non-prod Jenkins first with our real pipelines. Then I roll it out with `helm upgrade --atomic`, so if it fails health checks, it automatically rolls back.

</details>

---

### Q: The Jenkins admin credential is lost and the controller runs on Kubernetes with no external secret backup. What's the recovery path, and how do you prevent this from being a single point of failure again?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If it's still the first-time setup, the password is just sitting in a file you can read with `kubectl exec`. If it's a real lost admin account after setup, you have to shell in and reset it through a script, which isn't fun during an incident. Going forward, the real fix is hooking Jenkins up to the company's actual login system, so there's no single admin password to lose in the first place. We also back up the Jenkins data volume on a schedule, just in case.

</details>

---

### Q: Design a rollback strategy set that covers every layer of a deployment — application, infrastructure, and pipeline state.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Rollback isn't one thing, every layer needs its own plan. For the app in Kubernetes, it's `kubectl rollout undo`, instant, no rebuild. For Terraform, there's no real rollback command, so review the plan carefully before applying, since that's your real safety net. For source code, always use `git revert`, never `git reset --hard` on a shared branch, because that rewrites history everyone else already has.

</details>

---

## 4. Git — Version Control Internals

### Q: Explain the branching model you enforce, and specifically why `merge` vs `rebase` changes the risk profile of a shared branch.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**Merge** just adds a new commit and keeps history as it happened, so it's safe to use on shared branches. **Rebase** rewrites commit history with brand new IDs, and if you do that on a branch others already pulled, you break their copy too. So my rule is simple — rebase only on my own branch before I push it, never after it's shared. For day-to-day work, I lean toward short-lived feature branches merged straight into `main`.

</details>

---

### Q: How do you cleanly squash a range of commits before merging, and what breaks if you do it after pushing to a shared branch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use `git rebase -i HEAD~n` to squash a few messy commits into one clean commit. The big catch is if that branch is already shared, this creates new commit IDs and breaks everyone else's copy. So I always force-push with `--force-with-lease`, not a plain force, since that protects against overwriting someone else's work by mistake. And if it's truly shared already, I give people a heads-up first.

</details>

---

### Q: The `.git` directory is gone from a working copy. What's actually recoverable, and what isn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The `.git` folder holds your entire history, so losing it means losing everything that was never pushed anywhere. If there's a remote and everything was pushed already, it's easy — just re-clone or reset against the remote, nothing's actually lost. But any local commit that was never pushed is gone for good. In my experience, that's exactly why I push often and don't let work sit local for too long.

</details>

---

### Q: `git fetch` vs `git pull` — what's the actual difference in terms of working-tree risk?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git fetch` just downloads the latest changes, it doesn't touch your files at all, so it's completely safe. `git pull` is fetch plus an automatic merge right into your working copy, and if you have local changes that conflict, it can leave a mess. I use `pull` for everyday work, but never in scripts or automation without a safety flag like `--ff-only`, so it fails loudly instead of quietly merging something unexpected.

</details>

---

## 5. Terraform — State & Fundamentals

### Q: How do you bring an out-of-band-created AWS resource under Terraform management without recreating it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is `terraform import`, but the big catch is it only updates the state file, it does not write any config for you. If the code doesn't already match the real resource, the next plan will try to change or even destroy it. So my actual steps are — write the matching code first, run the import, then run `plan` and make sure it shows zero changes before I trust it.

</details>

---

### Q: What specifically breaks when Terraform state is kept local in a multi-engineer team, beyond "it's not backed up"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The real problem is there's no locking. If two people run `apply` at the same time, one just silently overwrites the other's changes, and now Terraform has no idea some resources even exist anymore. In my experience, a remote backend with locking, like S3 with a lock table, isn't optional for a team — it's required from day one. I also turn on versioning on that bucket, since that's the real way to roll back bad state.

</details>

---

### Q: What does a real Terraform testing/validation pipeline enforce before `apply` ever runs against production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform validate` only checks syntax, it has no idea if something's actually valid on AWS's side. So I layer checks — format check, then validate, then `tflint` for real provider-level issues, then a security scanner like `tfsec`, then a reviewed plan on the pull request. Production `apply` is always gated behind manual approval. The whole point is catching mistakes early, not at apply time.

</details>

---

## 6. Terraform — Multi-Account Architecture

### Q: Design the module/state topology for a Terraform codebase spanning dev/staging/prod across separate AWS accounts.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build shared, reusable modules for things like the VPC or the database, with no environment logic baked in. Then each environment gets its own thin folder that calls those modules with its own variables. The big thing is each environment also gets its own separate state file and backend, tied to its own account, so nothing overlaps. CI assumes a role into the right account before it ever touches anything.

</details>

---

### Q: How do you prevent a Terraform state read/write in one AWS account from ever touching another account's state, structurally rather than by convention?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't rely on people just remembering to point at the right config — that always breaks eventually. Instead, each account gets its own S3 bucket for state, and the bucket policy only allows that account's own CI role to touch it. So even a mistake in the pipeline config physically can't read or write another account's state. That's a real, structural wall, not just a rule on paper.

</details>

---

### Q: How do you keep secrets (DB credentials, API keys) out of both the Terraform codebase and the state file, given that Terraform state stores resource attributes in plaintext by default?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The big catch most people miss is that Terraform state is plain text by default, so even a secret passed in as a variable ends up sitting right there in the state file. My rule is secrets never go in as raw variables — they live in Secrets Manager or SSM, and Terraform just references them. On top of that, I turn on encryption on the state bucket and lock down who can even read it.

</details>

---

### Q: Trace exactly what happens end-to-end when a Terraform change merges to `main` in a CI/CD-driven multi-account pipeline.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, `terraform init` against the right backend. Then `plan`, and that output gets posted for a human to actually review — this is the real safety check. After manual approval on production specifically, the pipeline assumes the right account's role and runs `apply` using that exact saved plan, not a fresh one. Every log and plan gets kept for the audit trail, tied back to the commit that triggered it.

</details>

---

## 7. Terraform — Drift, Recovery & Advanced Ops

### Q: Someone manually changes a resource in the AWS console that Terraform manages. What does `terraform plan` actually show you, and how do you resolve it correctly versus incorrectly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform plan` will show a diff that reverts the console change back to what's in the code — that's totally normal, Terraform doesn't know or care who made the change. The big catch is you have to actually read that diff before applying. If someone made an emergency fix by hand during an outage, blindly applying would undo that fix. So I check first — if it was a real fix, I update the code to match it instead of reverting it.

</details>

---

### Q: How do you refactor a Terraform module that's already deployed in production without a destroy/recreate cycle on live resources?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Terraform tracks resources by their address in the code, not some hidden ID, so renaming things can make it think a resource was deleted and a new one created. I always version modules and test any refactor in non-prod first, checking the plan shows no surprise destroys. And if I genuinely need to rename something without touching the real resource, that's exactly what a `moved` block is for.

</details>

---

### Q: How do you structure Terraform to safely manage resources across multiple AWS regions from a single codebase?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One provider block only covers one region, so multi-region needs separate aliased provider blocks, like one for `us-east-1` and one for `eu-west-1`. The big catch is modules don't automatically know which provider to use — you have to pass it in explicitly. Skipping that step is exactly how people end up with a resource quietly deployed in the wrong region.

</details>

---

### Q: What's the actual access-control model that prevents an engineer from running `terraform apply` against production from their laptop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Telling people not to do it isn't a real control — it has to be enforced in IAM. The production deploy role can only be assumed by the CI pipeline's own identity, no human has a path to it at all. State bucket writes for production are locked the same way. I might allow read-only access for debugging, but nobody outside CI can ever actually apply.

</details>

---

### Q: How do you compose outputs from one Terraform stack (e.g., a VPC) as inputs to another (e.g., an ECS cluster) without merging them into one monolithic state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I split infra into layers — network, then compute, then data — so a mistake in one layer can't touch another's state. To connect them, I use `terraform_remote_state` as a read-only data source, so the ECS stack can read the VPC's subnet IDs without ever being able to change the VPC's state. The dependency only ever flows one direction, never back and forth.

</details>

---

### Q: A `terraform apply` fails halfway through, applying some resources and erroring on others. What's the actual recovery process?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The good news is this is totally recoverable. Terraform saves state as it goes, so anything that succeeded is already tracked correctly. I first figure out why it actually failed — usually a permissions or quota issue — fix that, then just re-run `apply`, and it only touches what's still left to do. If state itself ever looks wrong, I restore a previous version from the versioned backup instead of guessing.

</details>

---

## 8. AWS Infrastructure & Networking

### Q: An EC2 instance is unreachable. What's the deterministic order of checks, and why does that order matter?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always check this in the same order, so I don't waste time. Security group first, then the **NACL**, since that one needs both inbound and outbound rules allowed, unlike security groups. Then the route table, then the instance itself using a bastion or Session Manager, and only last, anything on the OS itself like a local firewall. Checking out of order just means chasing symptoms that aren't the real cause.

</details>

---

### Q: Design a multi-VPC network topology — what's the actual architectural decision between VPC Peering and Transit Gateway, and where does each break down at scale?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**VPC Peering** is one-to-one and doesn't chain — if A talks to B and B talks to C, A still can't reach C. That falls apart once you have more than a handful of VPCs. **Transit Gateway** solves that with one central hub everything connects through, but it costs more and becomes a bigger single point to worry about. In my experience, I use Peering for a small, stable set of VPCs, and move to Transit Gateway once I need real hub-style routing.

</details>

---

### Q: RDS performance degrades under load. What's the exact diagnostic sequence to isolate whether it's compute, connections, or query-level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

"RDS is slow" can mean three different things, so I check them in order. First CPU and memory — if that's maxed, it's a real capacity issue. Then connection count — if that's climbing, it's almost always the app not pooling connections properly, not the database itself. Then I check **Performance Insights** for slow queries — a missing index can look exactly like a capacity problem, but the fix there is an index, not a bigger instance.

</details>

---

### Q: What's the actual blast radius when a NAT Gateway fails, and how do you architect against it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A NAT Gateway only covers one availability zone. The big catch is if every private subnet across all zones points at just one NAT Gateway to save cost, losing that one zone kills outbound internet everywhere, not just that zone. In my experience, the right setup is one NAT Gateway per zone, so a failure only affects that single zone. It costs more, but it's worth it for production.

</details>

---

### Q: Walk through what actually changed to cut deployment cost by 40% — name the specific mechanisms, not the category.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

We moved workloads to EKS with autoscaling, so we stopped paying for capacity we didn't need around the clock. We added **Spot Instances** for workloads that could handle interruptions, which cut compute cost a lot on its own. We also slimmed down our Docker images, cleaned up unused resources like old volumes and load balancers through Terraform, and added S3 lifecycle rules to move old data to cheaper storage automatically.

</details>

---

## 9. ECS Fargate + ALB — Production Failure Modes

### Q: A Fargate service behind an ALB shows intermittent 502s for 2-3 minutes during every deployment. Diagnose the full failure chain and the fix.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is really three timers not lining up — how long the app takes to actually start, how long ECS waits before trusting a new task, and how long the ALB waits before sending it traffic. If ECS's wait time is too short, it kills the task before it's even ready. We fixed it by setting a proper `healthCheckGracePeriodSeconds`, using a real health check endpoint that only passes when the app is truly ready, and giving old tasks enough time to finish in-flight requests before they're removed.

</details>

---

### Q: Precisely distinguish `healthCheckGracePeriodSeconds` from ALB target group `deregistration_delay` — what does each actually control, and what happens if they're confused or misconfigured relative to each other?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`healthCheckGracePeriodSeconds` is about new tasks just starting up — it stops ECS from killing them too early. `deregistration_delay` is about old tasks shutting down — it gives the ALB time to stop sending them traffic first. If new tasks keep crash-looping on startup, that's the grace period. If you're seeing errors specifically when old tasks are removed, that's the deregistration delay. They fix two completely different moments in the deploy.

</details>

---

### Q: A container's real startup time (90s) exceeds the ALB's health check interval (30s). What's the exact ECS/ALB configuration to prevent 502s here, and why is a shorter interval alone not the fix?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The check interval just controls how often it polls, that part's fine on its own. The real risk is ECS's grace period being shorter than the actual 90 seconds the app needs to start — if it is, ECS kills the task before it ever gets a fair shot. So I set the grace period to something like 120 seconds, with real headroom, and make sure the health check only passes once the app is truly ready, not just running.

</details>

---

### Q: Given a 502 in production, how do you determine — from logs alone, without guessing — whether the ALB or the application generated it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A 502 just means the ALB didn't get a good response from the backend, it doesn't tell you why. I go straight to the ALB access logs and check the `target_status_code` field. A dash there means the ALB never even reached the app — that's infrastructure. A real number, like a `500`, means the app responded with an actual error — that's a code bug. That one field tells me exactly where to look next.

</details>

---

## 10. IAM, Secrets & Security Boundaries

### Q: How do you get secrets to a Lambda function at runtime without ever having them touch source control or unencrypted CI logs?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never go directly into config files that live in Git — even deleting them later doesn't remove them from history. They live in SSM or Secrets Manager instead, and get pulled in and decrypted right at deploy time. The Lambda's own role only gets access to the exact secret it needs, nothing broader. That way, even if the function were compromised, it couldn't read anything else.

</details>

---

### Q: How does a CI/CD pipeline authenticate into multiple AWS accounts without static, long-lived credentials sitting in the CI system?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Static access keys sitting in CI forever are a real risk, they don't expire and they're hard to revoke fast. Instead, I use **OIDC**, so CI requests short-lived credentials tied to a specific role, and that role only trusts that exact pipeline. From there, it assumes into each target account with permissions scoped tight to just that job. Every one of those actions gets logged, so there's a full trail if anything ever goes wrong.

</details>

---

### Q: What's the actual difference in engineering intent between SSM Parameter Store and Secrets Manager — when is using Parameter Store for a secret the wrong call?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Both can encrypt values, so that's not really the difference. Secrets Manager can automatically rotate credentials on a schedule, Parameter Store can't do that out of the box. So I use Parameter Store for config and secrets that never need rotating, and Secrets Manager for anything like a database password that genuinely should rotate regularly.

</details>

---

### Q: How do you structurally guarantee a feature-branch pipeline run can never deploy to production, rather than relying on pipeline YAML conditionals alone?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A branch check in the pipeline file alone isn't a real wall, since anyone with merge access could break it. So I enforce the real rule in the IAM trust policy itself — the production role only trusts tokens coming from the `main` branch specifically. Even if the pipeline config gets messed up, a feature branch's request to assume that role just gets rejected outright by AWS. That's the actual boundary that always holds.

</details>

---

## 11. Serverless — Lambda Architecture

### Q: What's the actual mechanism behind a Lambda cold start, and which architectural levers reduce it versus which just mask it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A cold start is the time it takes to spin up a brand new environment — download the code, start the runtime, and run anything outside your handler, like setting up a database client. That setup only happens once per environment, not every call, so the fix is moving that code outside the handler. For predictable, latency-sensitive traffic, **Provisioned Concurrency** keeps warm environments ready — but it doesn't help for a sudden, unpredictable spike bigger than what you provisioned.

</details>

---

### Q: Design a Lambda-based async processing pattern using SNS — what failure modes does this introduce that a synchronous call doesn't have, and how do you close them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

SNS can deliver the same message more than once, that's just how it works. So the big catch is your Lambda function has to be able to handle a duplicate message safely, otherwise you get duplicate actions, like two database rows instead of one. I always check a message ID against a small lookup table before actually processing it. I also add a **dead letter queue**, so anything that keeps failing doesn't just quietly disappear.

</details>

---

### Q: Design a single-codebase Serverless Framework deployment pipeline that deploys the same application across dev/stg/prod AWS accounts with clean environment-specific configuration and no secret leakage between environments.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One codebase, but config files per environment for anything non-sensitive, like a VPC ID. Secrets are never in those files at all — they get pulled from SSM or Secrets Manager at deploy time, scoped so each account can only read its own. The pipeline stage picks both the AWS account and the config together, so they can never drift apart. And we build the deployment package once and promote that exact same artifact through every environment.

</details>

---

## 12. Observability — Monitoring, Logging & Auditing

### Q: Design the logging/auditing layer for an AWS account — what does each service actually give you, and what's the gap if you only use one?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**CloudTrail** tells you who changed what. **VPC Flow Logs** tell you what traffic actually happened. **AWS Config** tells you what a resource's setup looked like over time. None of them alone gives the full picture — you need all three together, matched up by time, to really understand an incident. We ship all of it into one place so we can search across everything at once during an investigation.

</details>

---

### Q: A Redis cluster shows frequent evictions. What's the actual diagnostic path to determine whether this is a capacity problem or an application-design problem?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Evictions mean memory is full, but the reason matters. I check if memory usage keeps climbing forever with no leveling off — that usually means keys are being written without an expiry, which is an app bug, not a sizing problem. If usage is genuinely just large and steady, that's real undersizing, and I'd scale the cluster. I also always double check the eviction policy is actually set right for a cache, not something like `noeviction`.

</details>

---

### Q: What does a production-grade observability stack actually need to cover, beyond "we have CloudWatch"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CloudWatch alone can't show you a slow request as it travels across five different services — for that you need real **distributed tracing**, like New Relic or X-Ray. I also alert on things users actually feel, like error rate and slow response times, not just CPU usage, since those can miss real problems. And backups only count if you actually test restoring them, not just having them configured.

</details>

---

### Q: Define an incident response process that survives beyond "we look at the dashboard and fix it" — what's the actual structured loop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The loop is simple — detect it fast using real symptoms like error rate, analyze using logs and traces to find the actual cause, then fix the immediate problem first, even before the deep root cause is fully solved. After that, we always run a blameless postmortem with real action items, not just a report nobody follows up on. Without that last step, the same incident just comes back later.

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

The big catch is `-type f` — without it, folders could match too, and combined with `-delete`, that could wipe out whole directories by accident. In my experience, I always swap `-delete` for `-print` first and check the list before actually running the real thing, especially on a server I haven't touched before.

</details>

---

### Q: Write a script that counts ERROR occurrences in a log file, and explain why a naive substring match is a correctness risk at production log volume.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A simple check for the word "ERROR" anywhere in a line will also match things like "error_count field updated," which isn't a real error at all. That quietly gives you a wrong number with no warning. I always anchor the match to a proper word boundary, or better, check the actual log level field if the logs are structured. At real scale, this belongs in a log tool like CloudWatch Insights, not a small local script.

</details>

---

### Q: Automate Nginx installation across a fleet of 10 servers with Ansible — what's the idempotency guarantee the `apt` module gives you that a raw shell command doesn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A raw shell command just runs every single time, no matter what. The `apt` module checks the current state first, and only makes a change if it's actually needed — so running it again reports "ok," not "changed." I always use native modules like `apt` instead of raw shell commands, since that gives me a real signal when something unexpected actually changed.

</details>

---

### Q: What's the actual purpose of separating `roles/`, `group_vars/`, and `host_vars/` in an Ansible project, beyond directory tidiness?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This isn't just about tidiness, it's about control. **Roles** hold reusable automation that shouldn't change between projects. **`group_vars`** set defaults for a whole group of servers. **`host_vars`** override just one specific server, and always wins over the group setting. This setup is exactly what lets the same role run safely across dev, staging, and prod without changing the actual logic.

</details>

---

## 14. System Design — End-to-End Architecture

### Q: Describe a production multi-account AWS architecture end-to-end — the actual isolation boundaries, the deployment path, and where each control lives.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In my experience, the whole point of multi-account is keeping failures contained — a mistake in dev should never be able to touch prod. We run separate AWS accounts per environment, Lambda for event-driven work and ECS Fargate behind an ALB for containers, RDS and DynamoDB depending on the data pattern, and SQS and EventBridge to keep services loosely connected. Terraform is modular, with its own state per account, and deploys only happen through CI using short-lived roles, never from anyone's laptop. Production specifically needs manual approval, enforced at the IAM level, not just in the pipeline file, and secrets always come from SSM or Secrets Manager, never from source code.

</details>

---

<div align="center">

⭐ *If this helped you prep, consider starring the repo.*

</div>
