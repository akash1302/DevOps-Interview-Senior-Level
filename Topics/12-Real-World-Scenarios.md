# Senior DevOps Interview Questions: Real-World Scenarios

### Q: How would you troubleshoot a deployment that succeeded, but users are receiving 503 errors?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A successful pipeline only tells me the deployment completed, not that the app is actually healthy and able to receive traffic.

First I check where the 503 is coming from. If it's behind an ALB, I check the target group to see whether the new tasks or pods are actually healthy. For Kubernetes, I start with `kubectl get pods` and `kubectl describe pod`, then check the application logs. If a pod is running but not ready, I test the health endpoint directly from inside the pod.

For example, I've seen deployments where the app needed extra time to start because of database connections or cache warm-up. The container was running fine, but the readiness check kept failing, so traffic never got sent to it.

I also double-check the port — a mismatch between the Service, the target group, and what the container is actually listening on is a classic one I've hit after a Dockerfile change. If the previous version was healthy and this is customer-facing, I'll roll back while I keep investigating.

</details>

---

### Q: A terraform apply failed after provisioning half the infrastructure. How would you recover safely?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't immediately run another apply or destroy anything. First I look at the actual error to understand which resource failed and why — usually it's an IAM permission, an AWS quota, or a naming conflict.

Terraform updates state as resources succeed, so a lot of what completed is already tracked correctly. I run `terraform plan` to see what it thinks still needs to happen, and I'll check the AWS console too, since sometimes a resource did get created but Terraform failed while waiting on it.

Once I've fixed the root cause, I run `apply` again — it should only touch what's still outstanding. If I find something in AWS that isn't in state, I don't delete it, I bring it under management with `terraform import` instead.

I avoid `terraform destroy` unless there's a specific reason and I've actually reviewed what it's about to remove.

</details>

---

### Q: Your CI/CD pipeline is taking 25 minutes. How would you bring it down to under 5 minutes?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't start changing random steps. First I check the pipeline's stage timings to see what's actually taking most of the time — usually it's dependency installs, Docker builds, or tests.

For example, if every run is downloading dependencies from scratch, I'd add caching based on the lockfile so it's reused until dependencies actually change. I also look for steps running one after another that don't actually depend on each other — lint, unit tests, and some security checks can usually run in parallel instead.

For Docker builds, I'd enable layer caching and order the Dockerfile so a small code change doesn't rebuild everything. And I'd check whether heavy integration tests really need to run on every push, or just on merge to main.

After making changes, I measure the pipeline again rather than assuming it's faster.

</details>

---

### Q: How would you perform a zero-downtime Kubernetes cluster upgrade in production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't upgrade production first. I test the target version in a lower environment and check for deprecated APIs that might affect our workloads, ingress controllers, or other add-ons.

For EKS, I upgrade the control plane first, then handle worker nodes separately — never all at once. I'd create a new node group on the required version, move workloads over gradually, and remove the old nodes only once everything's stable.

Before draining anything, I make sure apps have proper readiness probes and PodDisruptionBudgets, then drain nodes in small batches and watch that pods come up healthy on the new ones. If anything starts behaving badly, I stop the rollout instead of continuing to the rest of the nodes.

</details>

---

### Q: How do you design a rollback strategy if the deployment stage itself fails?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

My first priority is making sure the existing healthy version keeps serving traffic. With proper readiness probes, Kubernetes won't send traffic to a new pod until it's actually ready, so the old version stays up the whole time.

I also set a rollout timeout in the pipeline — if the new version doesn't become healthy in time, the pipeline fails instead of waiting forever. I check the new pods, events, and logs to understand why it failed.

If it's already partly rolled out, `kubectl rollout undo deployment/<name>` gets back to the last known-good version quickly. What matters most to me is that rollback is a normal, tested operation — not something the team's figuring out for the first time during an actual incident.

</details>

---

### Q: Your application latency suddenly increased after a release. Walk me through your debugging approach.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>
First, I would check when the latency started and confirm whether it started immediately after the release.

I would check the ALB metrics and CloudWatch first, mainly target response time, request count, 4xx/5xx errors, CPU, and memory. This tells me whether the issue is at the load balancer, application, or infrastructure level.

Then I would check the application logs and look for slow API calls, timeout errors, connection errors, or any errors introduced by the new release.

If the application is waiting on the database, I would check RDS metrics such as CPU, connections, locks, and slow queries. I would also check the application's database connection pool because exhausted connections can cause requests to wait.

I would compare the current deployment with the previous version and check what code or configuration was changed.

If the new release is clearly causing the problem and the impact is high, I would rollback to the last stable version. Once the service is stable, I would reproduce the issue and find the actual root cause before deploying again.

So my approach is basically: check metrics → check logs → identify where the request is getting slow → compare the release changes → rollback if required → fix the root cause.


</details>

---

### Q: How would you manage secrets securely across multiple Kubernetes clusters and environments?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't keep production secrets in Git or directly in Kubernetes YAML files.

What I normally use is AWS Secrets Manager together with External Secrets Operator, so the manifest only holds a reference to the secret, and the actual value stays in Secrets Manager. Access is separated by environment too — the production workload can only read production secrets, staging can't touch them.

For EKS specifically, I use IAM roles for service accounts rather than putting AWS keys inside the pod. When a secret rotates in Secrets Manager, the Kubernetes copy refreshes automatically, and I make sure secrets never get printed in CI/CD logs or Terraform output by accident.

</details>

---

### Q: How do you investigate intermittent pod restarts when logs don't show any obvious errors?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the app's own logs are clean, I don't assume it actually crashed — I check what Kubernetes thinks happened first.

`kubectl describe pod` gives me the restart count, exit code, and recent events. If it's exit code 137 or OOMKilled, I check real memory usage against the configured limit. If instead it's a liveness probe failure, the app might not have crashed at all — Kubernetes killed it because the health check didn't respond in time, so I'd compare the probe's timeout against the app's real response time under load.

I also check when the restarts happen — during traffic spikes, scheduled jobs, or deploys — since that often points straight at the cause.

</details>

---

### Q: Your cloud bill increased by 40% this month. Where would you start your investigation?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd start with Cost Explorer, comparing the current period to the previous one, grouped by service. I don't start checking EC2, S3, and RDS randomly — this view tells me right away which service actually caused the jump.

If it's EC2, I check for new or forgotten instances and unexpected autoscaling activity. If it's data transfer, I check where the traffic's actually going — cross-region, internet egress, or between services. For S3, I check storage growth and whether a lifecycle policy quietly stopped working.

Once I find the actual resource, I confirm it with CloudTrail or the relevant metrics, fix the root cause, and set up a cost alert so the same thing gets caught earlier next time.

</details>

---

### Q: Explain the most challenging production incident you've handled and the architectural improvements you made afterward.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One incident I handled was the application slowing down because of a database issue. We started getting complaints, and checking the monitoring showed database connections climbing and some queries running much longer than normal.

As a temporary fix, we reduced the load and cleared unnecessary connections, then identified the actual problem query and worked with the dev team to fix it.

Afterward, we added better monitoring specifically for connection counts and query performance, so the same pattern would get caught earlier next time. The main thing I focused on during the incident itself was restoring the app first, and only then digging into the root cause.

</details>

---

### Q: If you had to redesign your current DevOps platform today, what would you do differently?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

My first focus would be standardization. I've worked in places where different apps were deployed in different ways — some through scripts, some through Helm, some manually — and that makes production support genuinely harder, since every app has a different troubleshooting process.

I'd standardize around Git-based workflows — Helm for packaging Kubernetes apps and a GitOps tool like ArgoCD for deployment, so the desired state lives in Git and deployments are consistent. I'd also standardize infrastructure through Terraform and get logging, monitoring, and alerting set up the same way everywhere.

I wouldn't change everything at once — I'd start with the biggest operational pain points and migrate gradually. The goal isn't using the newest tools, it's making deployments repeatable and troubleshooting predictable.

</details>

---
