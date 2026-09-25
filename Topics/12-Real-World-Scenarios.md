# Senior DevOps Interview Questions: Real-World Scenarios

### Q: How would you troubleshoot a deployment that succeeded, but users are receiving 503 errors?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A successful pipeline only tells me that the deployment completed. It doesn't mean the application is actually healthy and able to receive traffic.

First, I check where the 503 is coming from. If it's behind an AWS ALB, I check the target group and see whether the new ECS tasks or Kubernetes pods are healthy. If the targets are unhealthy, I check the health-check path, port, and the reason for the failure.

For Kubernetes, I usually start with kubectl get pods and kubectl describe pod, then check the application logs. If the pod is running but not ready, I test the health endpoint directly from the pod and verify the readiness probe configuration.

For example, I’ve seen deployments where the application needed some time to start because of database connections, migrations, or cache initialization. The container was running, but the ALB or Kubernetes readiness check was failing, so traffic wasn't being sent to it.

I also check whether the application is listening on the expected port and whether the ALB target group, Kubernetes Service, and container port are configured correctly. Then I compare the new deployment with the previous working revision, especially environment variables, secrets, configuration, and security-group rules.

So my usual flow is: check ALB/target health → check pod/task status → check health-check configuration → check application logs → verify port/configuration → compare with the last working deployment.

If the previous version was healthy, I would also consider rolling back while investigating, especially if it's a customer-facing production issue.

**Pipeline shows success → check target/pod health → unhealthy → curl the health endpoint directly from inside the pod → check app logs at that timestamp → usually a startup timing issue or a missed env var in the new revision.**

If health checks pass and 503s are still happening, I check if the port in the Service or target group actually matches what the container is listening on — a mismatched port after a Dockerfile change is a classic one I've hit.

</details>

---

### Q: A terraform apply failed after provisioning half the infrastructure. How would you recover safely?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If a Terraform apply fails halfway, I don't immediately run another apply or destroy anything. First, I look at the actual error and understand which resource failed and why.

Terraform updates the state as resources are successfully created or changed, so in many cases the resources that completed are already in the state. I first run `terraform plan` to see what Terraform thinks exists and what it still needs to create or change.

Before retrying, I also check the AWS console if needed, because sometimes the resource was actually created in AWS but Terraform failed while waiting for it or during a later step.

For example, if the failure was due to an IAM permission, AWS quota, dependency issue, or a resource that already exists, I fix that root cause first and then run `terraform plan` again.

If the plan looks correct, I run `terraform apply` again. Terraform should continue from the current state rather than recreate resources that are already managed.

If I find a resource in AWS that exists but is missing from Terraform state, I don't manually delete it. I verify whether it should be managed by Terraform and, if required, import it using `terraform import`.

I also check the Terraform state backend and locking if this is a shared environment, especially when we're using an S3 backend with locking. I make sure another pipeline or engineer isn't currently running Terraform.

So my approach is: **check the error → check state → run plan → verify AWS resources → fix the root cause → apply again → import anything that exists but isn't tracked.**

I avoid `terraform destroy` unless there is a specific reason and I've reviewed exactly what Terraform is going to remove.

</details>

---

### Q: Your CI/CD pipeline is taking 25 minutes. How would you bring it down to under 5 minutes?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, I wouldn't start changing random steps. I would check the pipeline execution time and identify which stages are actually taking most of the time.

Usually, dependency installation, Docker builds, tests, or security scans are the main contributors. For example, if every pipeline is downloading dependencies from scratch, I would add caching based on the lock file so the cache is reused until the dependencies change.

Then I look for steps that are running sequentially but don't actually depend on each other. Things like linting, unit tests, and some security checks can usually run in parallel.

For Docker builds, I would also enable Docker layer caching and make sure the Dockerfile is structured properly so that a small application change doesn't rebuild everything from scratch.

I would also check whether every test really needs to run on every developer push. Some heavy integration or end-to-end tests can run on merge to the main branch instead.

After making the changes, I measure the pipeline again rather than assuming it's faster.

So my approach is: *check stage timings → identify the bottleneck → add dependency/Docker caching → parallelize independent jobs → move heavy checks where appropriate → measure the result.*

</details>

---

### Q: How would you perform a zero-downtime Kubernetes cluster upgrade in production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For a production Kubernetes upgrade, I don't directly upgrade production first. I test the target Kubernetes version in a lower environment and check the version compatibility, deprecated APIs, workloads, ingress, controllers, and other add-ons.

For EKS, I normally upgrade the control plane first and then handle the worker nodes separately. I don't replace all nodes at once because that can create unnecessary risk.

For the worker nodes, I prefer creating the new node group with the required Kubernetes version, moving workloads gradually, and then removing the old nodes. If I'm using managed node groups or Karpenter, I can use that to make the node replacement more controlled.

Before draining nodes, I make sure the applications have proper readiness probes and PodDisruptionBudgets. Then I cordon and drain nodes in small batches and monitor whether the pods are coming up successfully on the new nodes.

I also monitor application health, ALB target health, pod restarts, error rates, and resource usage during the upgrade.

If something starts behaving badly, I stop the rollout instead of continuing with the remaining nodes.

So the basic approach is: *test the version → check deprecated APIs and dependencies → upgrade control plane → introduce new worker nodes → move workloads gradually → monitor → remove old nodes only after everything is stable.*

</details>

---

### Q: How do you design a rollback strategy if the deployment stage itself fails?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the deployment stage fails, my first priority is to make sure the existing healthy version is still serving traffic.

For a Kubernetes rolling deployment, I configure the deployment and readiness probes properly so Kubernetes doesn't send traffic to a new pod until it's actually ready.

I also set rollout timeouts in the CI/CD pipeline. If the new version doesn't become healthy within the expected time, the pipeline should fail instead of waiting indefinitely.

I then check the new pods, events, application logs, and readiness probe failures to understand why the rollout failed.

If the rollout has already partially progressed and the new version is causing issues, I can roll back using `kubectl rollout undo deployment/<name>` or deploy the previous known-good image.

After rollback, I verify the pods are healthy and check ALB/application metrics to make sure the customer impact has stopped.

The important thing for me is that rollback should be a normal, tested operation—not something we're trying to figure out for the first time during a production incident.

So the flow is: **keep the old version healthy → deploy the new version gradually → wait for readiness → detect failure → stop/rollback → verify service health → investigate the failed release.**

</details>

---

### Q: Your application latency suddenly increased after a release. Walk me through your debugging approach.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If latency increases immediately after a release, the new release is one of the first things I investigate. I compare the current version with the previous working version and check when the latency started.

First, I look at the monitoring or APM data to understand whether the increase is across all requests or only a specific API or endpoint. I also check p50, p95, and p99 because average latency can sometimes hide a problem affecting only a smaller percentage of requests.

Then I follow the request path. I check application CPU and memory, database response time, connection pools, slow queries, and any downstream or external API calls introduced or changed in the release.

For example, if a new code change starts making an additional database query for every request, or makes a synchronous call to another service, that can immediately increase response time under production traffic.

I also compare the application and infrastructure metrics before and after the deployment.

If the issue is clearly related to the new release and users are being affected, I don't spend 30 minutes trying to prove the exact root cause while the service is degraded. I roll back to the last known-good version first and then investigate the failed release with the pressure off.

So my approach is: *confirm when latency changed → identify affected endpoints → check p95/p99 → check app/DB/downstream dependencies → compare with previous release → rollback if needed → investigate the root cause.*

</details>

---

### Q: How would you manage secrets securely across multiple Kubernetes clusters and environments?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't keep production secrets directly in Git or inside Kubernetes YAML files.

For AWS environments, I would normally use AWS Secrets Manager and something like External Secrets Operator to make the secrets available to Kubernetes workloads.

The application manifest only contains the reference to the secret. The actual password, API key, or token stays in Secrets Manager.

I also separate access by environment. For example, the production workload should only have permission to read the production secrets it actually needs. The staging workload shouldn't have access to production secrets.

For EKS, I would use IAM roles for service accounts or the current EKS pod identity approach, depending on the platform setup, rather than putting AWS access keys inside the pod.

I also plan for rotation. When a secret changes in Secrets Manager, the Kubernetes secret should be refreshed automatically, and the application should be able to pick up the new value according to how the workload is configured.

I also make sure secrets don't get printed in CI/CD logs, Terraform output, application logs, or container environment dumps.

So the main approach is: *Secrets Manager → controlled IAM access → External Secrets → no secrets in Git → environment-level isolation → rotation and monitoring.*

</details>

---

### Q: How do you investigate intermittent pod restarts when logs don't show any obvious errors?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If application logs don't show anything, I don't assume the application crashed. I first check what Kubernetes thinks happened.

I start with `kubectl get pods` and `kubectl describe pod` and look at the restart count, container state, exit code, and Kubernetes events.

If I see exit code 137 or an OOMKilled status, I check the pod's memory usage and compare it with the configured memory limit. I also check whether the application itself has a memory issue or whether the container limit is simply too low.

If the event shows a liveness probe failure, then the application may not have actually crashed. Kubernetes may have restarted it because the health check wasn't responding within the configured timeout.

I would then check the probe configuration and compare it with the application's actual response time under load.

I also look at when the restarts happen. If they happen during traffic spikes, scheduled jobs, deployments, or when another workload on the same node is consuming resources, that gives me another direction.

So my flow is: *check restart count → describe the pod → check exit code/events → check OOM and resource usage → check liveness/readiness probes → correlate restart time with traffic and node activity.*

</details>

---

### Q: Your cloud bill increased by 40% this month. Where would you start your investigation?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

SI would start with AWS Cost Explorer and compare the current period with the previous period, grouped by service and account/environment.

The first thing I want to know is which service actually caused the increase. I don't start checking EC2, S3, RDS, and everything else randomly.

If EC2 is the main increase, I check for new instances, forgotten instances, autoscaling activity, and environments that were supposed to be temporary.

If the increase is in data transfer, I investigate where the traffic is going—cross-region traffic, internet egress, NAT Gateway usage, or traffic between services.

For S3, I would check storage growth, request costs, data transfer, and lifecycle policies.

For ECS or EKS, I check whether workloads scaled unexpectedly or whether resource requests/limits caused more capacity to run than expected.

Once I identify the actual resource causing the increase, I verify it with CloudTrail or the relevant service metrics and then fix the root cause.

I also add cost alerts or anomaly detection so the same type of increase is detected earlier.

So my approach is: *Cost Explorer → identify the service → identify the actual resource/cost driver → check recent infrastructure changes → verify with metrics/CloudTrail → fix → add an alert to prevent recurrence.*

</details>

---

### Q: Explain the most challenging production incident you've handled and the architectural improvements you made afterward.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

One production issue I handled was an application becoming slow because of a database issue.

We started getting complaints that the application was responding slowly. I checked the application and AWS monitoring and found that the database connections were getting high and some queries were taking longer than normal.

I checked the database connections, application logs, and slow queries to find the issue.

As a temporary fix, we reduced the load and cleared the unnecessary connections. Then we identified the query causing the problem and worked with the development team to fix it.

After that, we added better monitoring for database connections and query performance so we could identify the same issue earlier.

The main thing I focused on during the incident was first restoring the application, and then finding and fixing the root cause.

</details>

---

### Q: If you had to redesign your current DevOps platform today, what would you do differently?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If I were redesigning the platform today, my first focus would be standardization.

I've worked in environments where different applications were deployed in different ways—some through scripts, some through CI/CD, some through Helm, and some manually. That makes production support harder because every application has a different deployment and troubleshooting process.

I would standardize the deployment process around Git-based workflows. For Kubernetes workloads, I would consider Helm for packaging and ArgoCD or another GitOps approach for deployment, so the desired state is stored in Git and deployments are consistent.

I would also standardize infrastructure through Terraform, CI/CD pipelines, secrets management, logging, monitoring, and alerting.

The second area I'd improve is observability. I want metrics, centralized logs, dashboards, alerts, and tracing available from the beginning, especially when applications start communicating with multiple services.

I wouldn't try to change everything at once. I'd first identify the biggest operational problems, standardize the common pieces, and then migrate applications gradually.

For me, the goal isn't to introduce tools just because they're popular. The goal is to make deployments repeatable, troubleshooting easier, and production operations more predictable.

</details>

---
