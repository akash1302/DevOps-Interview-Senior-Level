<div align="center">

# 🚀 DevOps Interview Q&A — Senior Level

![Topics](https://img.shields.io/badge/topics-14-blue)
![Level](https://img.shields.io/badge/level-Senior-orange)
![Format](https://img.shields.io/badge/format-Markdown-informational)
![Stack](https://img.shields.io/badge/stack-AWS%20%7C%20K8s%20%7C%20Terraform%20%7C%20CI%2FCD-success)

A consolidated, topic-wise interview reference built from a general DevOps cheat sheet, an AWS/Infra troubleshooting set, and an advanced Serverless/ECS/Terraform architecture guide.

*Click any question to expand its answer.*

</div>

---

## 📚 Table of Contents

| # | Topic | # | Topic |
|---|-------|---|-------|
| 1 | [🐳 Docker](#-docker) | 8 | [☁️ AWS Infra & Networking](#️-aws-infrastructure--networking) |
| 2 | [☸️ Kubernetes](#️-kubernetes) | 9 | [⚖️ ECS Fargate + ALB Troubleshooting](#️-ecs-fargate--alb-troubleshooting) |
| 3 | [🔧 Jenkins & CI/CD](#-jenkins--cicd) | 10 | [🔐 Secrets, IAM & Security](#-secrets-iam--security) |
| 4 | [🔀 Git](#-git) | 11 | [⚡ Serverless (Lambda)](#-serverless-lambda) |
| 5 | [🌍 Terraform — Fundamentals](#-terraform--fundamentals) | 12 | [📊 Monitoring, Logging & Auditing](#-monitoring-logging--auditing) |
| 6 | [🏢 Terraform — Multi-Account](#-terraform--multi-account--multi-environment) | 13 | [🐧 Linux / Scripting](#-linux--scripting) |
| 7 | [🧠 Terraform — Advanced / Drift & Ops](#-terraform--advanced--drift--ops) | 14 | [🏗️ Architecture & Behavioral](#️-architecture--behavioral) |

---

## 🐳 Docker

<details>
<summary><b>Q: What are the types of Docker volumes?</b></summary><br>

Three types:
1. **Host Volume** – maps a directory from the host machine to the container.
2. **Anonymous Volume** – Docker manages the volume without a specific name.
3. **Named Volume** – user-defined and persisted independently of containers.
</details>

<details>
<summary><b>Q: Difference between CMD and ENTRYPOINT?</b></summary><br>

- `CMD`: default command, can be overridden.
- `ENTRYPOINT`: always executed, cannot be overridden (unless using `--entrypoint`).
</details>

<details>
<summary><b>Q: Docker container stops & restarts — data is lost. How do you fix it?</b></summary><br>

Use Docker **volumes** or **bind mounts** to persist data outside the container's writable layer.
</details>

<details>
<summary><b>Q: How do you delete unused Docker containers and images?</b></summary><br>

```bash
docker system prune -a
```
</details>

---

## ☸️ Kubernetes

<details>
<summary><b>Q: What is Kubernetes taint and toleration?</b></summary><br>

- **Taint** — applied to a node to restrict pod scheduling.
- **Toleration** — applied to a pod to allow it to run on tainted nodes.

Used to isolate workloads (e.g., dedicate nodes to specific apps).
</details>

<details>
<summary><b>Q: Can pod-to-pod communication happen by default?</b></summary><br>

Yes — the cluster network is flat and non-restrictive by default, unless NetworkPolicies restrict it.
</details>

<details>
<summary><b>Q: How can you restrict pod-to-pod communication?</b></summary><br>

Using Kubernetes **NetworkPolicies** — deny all traffic by default, then allow only specific namespace/app/label combinations.
</details>

<details>
<summary><b>Q: What are Helm charts?</b></summary><br>

A Helm chart is a package of YAML templates used to deploy Kubernetes applications through versioned, repeatable, configurable deployments.
</details>

<details>
<summary><b>Q: Helm chart folder structure?</b></summary><br>

```
Chart.yaml
values.yaml
templates/
  deployment.yaml
  service.yaml
  ingress.yaml
  configmap.yaml
  pvc.yaml
charts/
README.md
```
</details>

<details>
<summary><b>Q: Difference between StatefulSet and Deployment?</b></summary><br>

| | Deployment | StatefulSet |
|---|---|---|
| Use case | Stateless apps | Stateful apps (DB, Kafka) |
| Identity | No stable identity | Stable network identity |
| Storage | Shared/none | Persistent storage per pod |
</details>

<details>
<summary><b>Q: Explain RBAC in Kubernetes.</b></summary><br>

Role-Based Access Control — governs who can access what, using **Roles**, **ClusterRoles**, **RoleBindings**, and **ClusterRoleBindings**.
</details>

<details>
<summary><b>Q: How do you store secrets in Kubernetes?</b></summary><br>

- Kubernetes Secrets (Base64 encoded)
- AWS Secrets Manager + CSI driver
- HashiCorp Vault
</details>

<details>
<summary><b>Q: What is CrashLoopBackOff?</b></summary><br>

A pod keeps crashing and Kubernetes keeps restarting it.

> [!NOTE]
> Common causes: app errors, wrong configs, missing dependencies, liveness probe failures.
</details>

<details>
<summary><b>Q: Application deployed in EKS but not accessible externally — how do you debug?</b></summary><br>

1. Check Service type (LoadBalancer / NodePort).
2. Check Ingress configuration.
3. Check Security Group inbound rules.
4. Check NACLs / VPC routing.
5. Check DNS mapping.
6. Check pods are running and service endpoints exist.
</details>

<details>
<summary><b>Q: How does communication happen inside an EKS cluster?</b></summary><br>

Through the CNI plugin (Kubernetes network), Services (ClusterIP/NodePort/LoadBalancer), CoreDNS, and VPC routing.
</details>

<details>
<summary><b>Q: Where do you deploy microservices?</b></summary><br>

Kubernetes (EKS), Docker containers, ECS, EC2, or Fargate.
</details>

---

## 🔧 Jenkins & CI/CD

<details>
<summary><b>Q: How would you deploy Jenkins in your organization?</b></summary><br>

Four common approaches: EC2 install (manual), Docker container, Helm chart on EKS, or raw Kubernetes YAML manifests. Commonly EKS + Helm for scalability.
</details>

<details>
<summary><b>Q: How long does a Jenkins job take to complete?</b></summary><br>

Depends on pipeline stages, build time, tests, and infra speed — typically 1 to 15+ minutes.
</details>

<details>
<summary><b>Q: Jenkins pipeline fails — how do you debug?</b></summary><br>

- Check console output
- Check agent availability
- Validate credentials
- Check Docker build errors
- Check Git authentication
- Check stage-specific errors
</details>

<details>
<summary><b>Q: What types of Jenkins agents have you used?</b></summary><br>

Static EC2 agents, dynamic agents via the Kubernetes plugin, Docker agents, self-hosted runners.
</details>

<details>
<summary><b>Q: Jenkins deployed via Helm — how do you update plugins?</b></summary><br>

1. Update the plugin list in `values.yaml`.
2. Upgrade the chart:
```bash
helm upgrade jenkins -f values.yaml jenkins/jenkins
```
</details>

<details>
<summary><b>Q: Lost the Jenkins admin password — how do you restore it?</b></summary><br>

```bash
kubectl exec -it <pod> -- cat /var/jenkins_home/secrets/initialAdminPassword
```
Or reset via the admin account, or restore from backup.
</details>

<details>
<summary><b>Q: What Jenkins plugins do you commonly use?</b></summary><br>

Git, Pipeline, Credentials, Blue Ocean, Docker, Kubernetes, Slack, SonarQube.
</details>

<details>
<summary><b>Q: Where do you store Jenkinsfiles and Dockerfiles?</b></summary><br>

Typically per-service, inside each repository (`/app/Dockerfile`, `/app/Jenkinsfile`).
</details>

<details>
<summary><b>Q: What is your CI/CD approach?</b></summary><br>

Automated pipeline with build → test → deploy stages plus approval gates.
</details>

<details>
<summary><b>Q: What deployment strategies do you use?</b></summary><br>

Rolling update, Blue-Green, Canary, Recreate.
</details>

<details>
<summary><b>Q: What rollback strategies do you follow?</b></summary><br>

Helm rollback, Kubernetes deployment revision rollback, EC2 AMI rollback, Terraform manual state revert, Git revert.
</details>

---

## 🔀 Git

<details>
<summary><b>Q: Explain your Git branching strategy.</b></summary><br>

Gitflow is most common:
- `main`/`master` → stable production
- `develop` → active development
- `feature/` → new features
- `release/` → pre-production
- `hotfix/` → quick patches on production
</details>

<details>
<summary><b>Q: Difference between git clone vs git fork, merge vs rebase?</b></summary><br>

- **Clone**: copies the repo locally.
- **Fork**: copies the repo under your own account.
- **Merge**: combines changes with a merge commit.
- **Rebase**: rewrites commit history for a cleaner, linear log.
</details>

<details>
<summary><b>Q: How do you combine multiple commits into a single commit?</b></summary><br>

```bash
git rebase -i HEAD~n
```
</details>

<details>
<summary><b>Q: What is the .git folder?</b></summary><br>

Stores repo history, branches, objects, and configuration.
</details>

<details>
<summary><b>Q: If you lose the .git folder, how do you restore it?</b></summary><br>

You can't fully restore it — reinitialize instead:
```bash
git init
git remote add origin <url>
git fetch
```
</details>

<details>
<summary><b>Q: Difference between git pull and git fetch?</b></summary><br>

- `git fetch` → downloads changes but doesn't merge.
- `git pull` → downloads and merges automatically.
</details>

---

## 🌍 Terraform — Fundamentals

<details>
<summary><b>Q: How do you add an existing resource to the Terraform state file?</b></summary><br>

```bash
terraform import <resource_address> <resource_id>
```
</details>

<details>
<summary><b>Q: What problems occur if Terraform state is stored locally in a team environment?</b></summary><br>

- No shared view of infrastructure state
- High risk of state drift between users
- No locking → concurrent applies can corrupt infrastructure
- No backup or version history
- Difficult to audit changes

> [!IMPORTANT]
> Remote state with locking is mandatory for team-based Terraform usage.
</details>

<details>
<summary><b>Q: What is your Terraform testing strategy?</b></summary><br>

- `terraform validate` for syntax
- `tflint` for linting
- `tfsec` for security scanning
- Plan review in merge requests
- Apply first in non-prod environments
</details>

---

## 🏢 Terraform — Multi-Account / Multi-Environment

<details>
<summary><b>Q: How do you structure Terraform for multi-account, multi-environment deployments?</b></summary><br>

Modular design: core components (VPC, ECS, RDS, EFS, IAM) as reusable modules; each environment (dev/stg/prod) has a thin root module passing environment-specific variables. Remote state in S3 with DynamoDB locking; CI/CD assumes roles into target accounts before applying.
</details>

<details>
<summary><b>Q: How do you handle Terraform state across multiple AWS accounts?</b></summary><br>

Separate S3 bucket and DynamoDB lock table per account/environment for state isolation. IAM restricts state access to CI/CD roles only.
</details>

<details>
<summary><b>Q: How do you manage secrets in Terraform without exposing them in state files?</b></summary><br>

- Never hardcode secrets in variables/tfvars
- Store in AWS Secrets Manager or SSM Parameter Store (SecureString)
- Reference dynamically via Terraform data sources
- Restrict state file access via IAM; encrypted remote state storage
</details>

<details>
<summary><b>Q: How do Terraform deployments run through CI/CD?</b></summary><br>

`terraform init` (remote backend) → `terraform plan` (reviewed) → manual approval for prod → `terraform apply` after AssumeRole into the target account. Plans/artifacts stored for audit.
</details>

<details>
<summary><b>Q: What happens if two team members run terraform apply at the same time?</b></summary><br>

DynamoDB state locking prevents concurrent applies — the second run waits or fails until the lock releases.
</details>

<details>
<summary><b>Q: How do you roll back Terraform changes?</b></summary><br>

No direct rollback command, but: restore a previous state version from S3, re-apply a previous code version, or use plan approvals to catch issues pre-apply.
</details>

<details>
<summary><b>Q: How do you handle Terraform module versioning?</b></summary><br>

Version modules with Git tags or commit hashes; root modules pin to fixed versions to avoid breaking changes.
</details>

<details>
<summary><b>Q: How do you avoid hardcoding environment values?</b></summary><br>

Use variables and environment-specific `.tfvars` files; CI/CD injects the correct file per environment.
</details>

<details>
<summary><b>Q: What is your strategy for structuring reusable multi-environment infra safely?</b></summary><br>

Modular design per component (VPC, RDS, ECS, EFS, S3), thin environment layers, remote backend in S3 + DynamoDB locking, state file versioning, code review, separate state per environment, least-privilege IAM on the backend.
</details>

---

## 🧠 Terraform — Advanced / Drift & Ops

<details>
<summary><b>Q: How do you handle Terraform drift if someone manually changes a resource in AWS?</b></summary><br>

`terraform plan` detects drift by comparing desired vs. actual state. Policy requires production changes go through Terraform; on drift, either revert via `terraform apply` or `terraform import` the change if intentional.
</details>

<details>
<summary><b>Q: How do you safely refactor a Terraform module already in production use?</b></summary><br>

Version the module with Git tags, introduce changes in a new version, validate in non-prod first, then update the production root module to the new version.
</details>

<details>
<summary><b>Q: How do you manage Terraform across different AWS regions?</b></summary><br>

Multiple `provider` blocks with region aliases; modules accept provider aliases for cross-region deployment from a single project.
</details>

<details>
<summary><b>Q: How do you control who can run terraform apply on production?</b></summary><br>

Only CI/CD pipeline roles can access production remote state and assume production deployment roles. Manual `apply` from laptops is blocked via IAM, plus manual approval gates for prod.
</details>

<details>
<summary><b>Q: How do you pass outputs from one Terraform stack to another?</b></summary><br>

Use the `terraform_remote_state` data source to read outputs from another stack's state (e.g., VPC → ECS → RDS layering).
</details>

<details>
<summary><b>Q: Have you used Terraform workspaces? Why or why not?</b></summary><br>

Prefer separate state files per environment over workspaces for clearer isolation and simpler access control in multi-account setups. Workspaces suit same-account multi-env better.
</details>

<details>
<summary><b>Q: What happens if the Terraform state file is accidentally deleted?</b></summary><br>

Restore the last version from S3 versioning. If fully lost, resources can be reconciled via `terraform import`, but versioned backups should prevent this.
</details>

<details>
<summary><b>Q: If one terraform apply fails halfway, how do you recover safely?</b></summary><br>

Terraform is state-aware — successfully created resources are already recorded. Fix the root cause and re-run `apply`; Terraform computes the delta. If state is inconsistent, use `terraform state rm` or `terraform import` to reconcile, or roll back via versioned state.
</details>

<details>
<summary><b>Q: How do you implement Terraform in CI/CD without exposing AWS credentials?</b></summary><br>

STS AssumeRole with short-lived credentials — no static AWS keys stored in the CI system; IAM trust policies restrict which pipelines can assume deployment roles.
</details>

<details>
<summary><b>Q: How do you handle long-running Terraform applies?</b></summary><br>

Run in CI/CD runners with extended job timeouts; for critical changes, run `plan` first, review, then `apply`. State locking prevents parallel-execution conflicts.
</details>

---

## ☁️ AWS Infrastructure & Networking

<details>
<summary><b>Q: EC2 instance not accessible — what do you check?</b></summary><br>

Security group, NACL, route table, instance status; try access via bastion host; verify SSH key and OS-level firewall.
</details>

<details>
<summary><b>Q: How do you design a multi-VPC architecture?</b></summary><br>

CIDR planning, separate VPCs per environment/purpose, connect via VPC Peering or Transit Gateway, ensure correct routing.
</details>

<details>
<summary><b>Q: How do you secure a production environment?</b></summary><br>

Private subnets, bastion host, IAM roles, restricted security groups, KMS encryption, no public exposure.
</details>

<details>
<summary><b>Q: What happens if the NAT Gateway fails?</b></summary><br>

Private instances lose internet access — mitigate by deploying NAT Gateways across multiple AZs.
</details>

<details>
<summary><b>Q: RDS performance degradation — how do you fix it?</b></summary><br>

Check CPU, connection count, slow query logs; optimize queries, add indexes, or scale the instance.
</details>

<details>
<summary><b>Q: How do you ensure high availability?</b></summary><br>

Multi-AZ deployment, ALB, Auto Scaling Groups, RDS failover, redundancy at every layer.
</details>

<details>
<summary><b>Q: How do you implement Auto Scaling?</b></summary><br>

CloudWatch alarms on CPU or request count trigger Auto Scaling Group policies.
</details>

<details>
<summary><b>Q: How do you handle a disk-full issue?</b></summary><br>

Clean and rotate logs, increase volume size.
</details>

<details>
<summary><b>Q: What EC2 instance types have you used, and are they sufficient?</b></summary><br>

Example: t3.medium / t3.large, sized against CPU, memory, auto-scaling needs, and application load.
</details>

<details>
<summary><b>Q: What load balancers have you used?</b></summary><br>

ALB, NLB, CLB; in Kubernetes — Ingress Controller and Service type LoadBalancer.
</details>

<details>
<summary><b>Q: What are the S3 storage classes?</b></summary><br>

Standard, Standard-IA, One Zone-IA, Glacier, Glacier Deep Archive, Intelligent-Tiering.
</details>

<details>
<summary><b>Q: How do you optimize AWS cost?</b></summary><br>

Right-sizing, reserved instances, lifecycle policies, log optimization.
</details>

<details>
<summary><b>Q: How did you reduce deployment cost by 40%?</b></summary><br>

Migrated workloads to EKS with auto-scaling, used Spot instances, implemented CI/CD to cut idle compute, optimized Docker images, removed unused infrastructure via Terraform cleanup, applied S3 lifecycle policies.
</details>

---

## ⚖️ ECS Fargate + ALB Troubleshooting

<details>
<summary><b>Q: ALB returning 5xx errors — how do you troubleshoot?</b></summary><br>

Check ALB metrics (5xx count) and target group health, log into the EC2/task and check app logs, port binding, CPU/memory, and review recent deployments.
</details>

<details>
<summary><b>Q: Node.js app on ECS Fargate behind an ALB — intermittent 502s for 2–3 minutes during deployment. How do you design and troubleshoot this?</b></summary><br>

**Possible root causes**
- Target registration delay — new tasks not yet passing health checks while old tasks are drained
- Misconfigured health checks (wrong path, port mismatch, insufficient grace period)
- Application startup delay — ALB routes traffic before the app is ready
- Security Group misconfiguration between ALB and ECS task
- ALB idle timeout too low for long-running requests
- Insufficient deregistration delay dropping in-flight requests

**How to investigate**
- Check ALB target group health status
- Review ECS service events for task replacement timing
- Inspect ALB access logs to confirm the 502 source
- Check CloudWatch container logs for startup time/errors
- Access the container directly via its ENI, bypassing the ALB
- Verify Security Group rules: ALB SG → ECS SG

**How to prevent going forward**
- Configure ECS health check grace period appropriately
- Use target group deregistration delay
- Implement a `/health` readiness endpoint that only passes once fully ready
- Enable rolling deployments with minimum healthy percent > 100
- Tune ALB idle timeout for long requests
- Confirm correct SG rules on the container port
</details>

<details>
<summary><b>Q: Difference between ALB health check grace period and target group deregistration delay?</b></summary><br>

| | Health Check Grace Period | Deregistration Delay |
|---|---|---|
| Configured in | ECS Service | ALB Target Group |
| Purpose | Ignore ALB health check failures for X seconds after a new task starts | Keep sending in-flight requests to a draining task for X seconds before stopping |
| Protects against | Premature task replacement | Dropped connections during deploys |
</details>

<details>
<summary><b>Q: Container takes 90 seconds to start, but ALB health checks run every 30 seconds — how do you configure ECS to avoid 502s?</b></summary><br>

Set the ECS health check grace period to at least 90–120 seconds, and ensure the health check path only returns success once the app is fully ready.
</details>

<details>
<summary><b>Q: Where do you check logs to confirm if a 502 is from ALB or the application?</b></summary><br>

- ALB access logs (S3) show whether ALB generated the 502
- CloudWatch container logs show if the app itself errored
- Target group health status shows failing health checks

> [!TIP]
> `target_status_code = -` in ALB logs → ALB-side error. A code like `500` → app-side error.
</details>

<details>
<summary><b>Q: What is blue/green deployment in ECS?</b></summary><br>

Two identical environments where traffic shifts from the old version (blue) to the new version (green) for zero-downtime releases.
</details>

<details>
<summary><b>Q: How do you roll back a failed Serverless deployment?</b></summary><br>

Use `serverless rollback`, or redeploy the last stable CloudFormation stack version.
</details>

---

## 🔐 Secrets, IAM & Security

<details>
<summary><b>Q: How do you prevent sensitive values (DB passwords, API tokens) from being committed to Git while still making them available to Lambda at runtime?</b></summary><br>

Store secrets in SSM Parameter Store (SecureString) or Secrets Manager with KMS encryption. Serverless framework references them directly in `serverless.yml` during deployment so Lambda receives them as environment variables. IAM restricts read access to only the specific Lambda execution role.
</details>

<details>
<summary><b>Q: How does your CI/CD pipeline authenticate into multiple AWS accounts securely?</b></summary><br>

Pipeline runs in a centralized DevOps account and uses STS AssumeRole to temporarily assume a deployment role in each target account (dev/stg/prod), each with least-privilege permissions. Short-lived credentials only — no long-term keys. CloudTrail logs all role assumptions for auditability.
</details>

<details>
<summary><b>Q: How do you implement secure cross-account access generally?</b></summary><br>

IAM roles with STS AssumeRole and trust relationships; avoid static credentials.
</details>

<details>
<summary><b>Q: How do you manage secrets (e.g., RDS passwords) generally?</b></summary><br>

SSM Parameter Store and Secrets Manager, both with KMS encryption.
</details>

<details>
<summary><b>Q: Difference between SSM Parameter Store and Secrets Manager?</b></summary><br>

Parameter Store is for general configuration and secrets with basic encryption; Secrets Manager is purpose-built for managing, rotating, and auditing sensitive secrets.
</details>

<details>
<summary><b>Q: How do you ensure a feature branch deployment can't accidentally hit production?</b></summary><br>

- Git branch rules: only `main`/`release` trigger production pipelines
- Environment protection requiring manual approval for prod
- Pipeline rules restrict feature branches to ephemeral/dev environments
- Production IAM roles can only be assumed by approved pipeline jobs
</details>

---

## ⚡ Serverless (Lambda)

<details>
<summary><b>Q: What causes Lambda cold starts?</b></summary><br>

AWS needs to initialize a new execution environment, typically after inactivity or during scaling events.
</details>

<details>
<summary><b>Q: Explain a Lambda + SNS use case.</b></summary><br>

Lambda processes events and triggers SNS alerts or downstream workflows.
</details>

<details>
<summary><b>Q: How do you design a CI/CD pipeline to deploy the same Serverless app to multiple AWS accounts with clean, environment-specific configs?</b></summary><br>

Single codebase for all environments; a `config/` directory holds non-sensitive per-environment values (VPC IDs, subnet/SG IDs, ARNs, domains, feature flags). Secrets stay out of Git, pulled from SSM/Secrets Manager at deploy time. Reusable GitLab CI templates per environment inject variables, select the AWS account, and load the right config. STS AssumeRole handles multi-account deploys. Artifacts are built once in S3/ECR and promoted across environments for reproducibility.
</details>

---

## 📊 Monitoring, Logging & Auditing

<details>
<summary><b>Q: How do you handle logging and auditing?</b></summary><br>

CloudTrail for API-level logs, AWS Config for change tracking, VPC Flow Logs for network visibility.
</details>

<details>
<summary><b>Q: How do you implement monitoring?</b></summary><br>

CloudWatch metrics + alarms + SNS notifications across EC2, ALB, RDS, Redis.
</details>

<details>
<summary><b>Q: Redis eviction happening frequently — what's your action?</b></summary><br>

Check memory usage via CloudWatch, increase node size or tune the eviction policy (e.g., LRU), and validate application-side caching logic.
</details>

<details>
<summary><b>Q: What monitoring/observability stack do you use in production?</b></summary><br>

New Relic APM for Lambda/ECS performance monitoring and distributed tracing/error tracking; CloudWatch Logs and Metrics for infrastructure-level monitoring; Slack alerts on latency/error threshold breaches.
</details>

<details>
<summary><b>Q: What is your incident handling approach?</b></summary><br>

**Detect → Analyze → Fix → RCA → Prevent recurrence**
</details>

<details>
<summary><b>Q: What is your backup strategy?</b></summary><br>

RDS automated backups, EBS snapshots, lifecycle policies, and periodic restore testing.
</details>

---

## 🐧 Linux / Scripting

<details>
<summary><b>Q: How do you locate the path of a file?</b></summary><br>

```bash
find / -name filename
```
</details>

<details>
<summary><b>Q: How do you delete log files larger than 50MB and older than 30 days?</b></summary><br>

```bash
find /var/log -type f -size +50M -mtime +30 -delete
```
</details>

<details>
<summary><b>Q: Write a Python program to read a log file and print the error count.</b></summary><br>

```python
count = 0
with open("app.log", "r") as f:
    for line in f:
        if "ERROR" in line:
            count += 1
print("Total ERROR messages:", count)
```
</details>

<details>
<summary><b>Q: Install Nginx on 10 servers using Ansible.</b></summary><br>

Inventory + playbook, e.g.:
```yaml
hosts: web
tasks:
  - name: Install nginx
    apt: name=nginx state=present
```
</details>

<details>
<summary><b>Q: Explain the Ansible project structure.</b></summary><br>

```
inventories/
roles/
playbooks/
group_vars/
host_vars/
ansible.cfg
```
</details>

<details>
<summary><b>Q: What is an Ansible playbook?</b></summary><br>

A YAML file that defines tasks, modules, and roles to execute automation.
</details>

---

## 🏗️ Architecture & Behavioral

<details>
<summary><b>Q: Who manages infrastructure in your organization?</b></summary><br>

The DevOps team, using IaC tools like Terraform, CloudFormation, and Ansible.
</details>

<details>
<summary><b>Q: What is your project architecture (high level)?</b></summary><br>

Microservices in EKS, CI/CD via Jenkins/GitHub Actions, IaC via Terraform, monitoring via Prometheus & Grafana, images in ECR, logs in CloudWatch.
</details>

<details>
<summary><b>Q: How do you receive tickets/work?</b></summary><br>

JIRA, ServiceNow, or Azure DevOps Boards.
</details>

<details>
<summary><b>Q: How many DevOps engineers are on your team?</b></summary><br>

Typical answer: 4–6, depending on project size.
</details>

<details open>
<summary><b>Q: Explain your current infrastructure architecture end-to-end (senior-level answer)</b></summary><br>

Multi-account AWS setup (dev, dev2, staging, UAT, production) for strong isolation:
- Backend APIs/event-driven services on Lambda via Serverless Framework
- Containerized frontend/backend on ECS Fargate
- API Gateway and ALB as entry points
- RDS PostgreSQL for relational data, DynamoDB for key-value use cases
- S3 for artifacts/static content
- SQS and EventBridge for async processing
- EFS for shared filesystem needs
- Terraform (modular) provisions everything; remote state in S3 with DynamoDB locking

**CI/CD flow**
1. Code push triggers the pipeline
2. Build and package artifacts (Lambda zip or Docker image)
3. Store immutable artifacts in S3/ECR
4. STS AssumeRole into the target AWS account
5. Apply infra changes via Terraform or Serverless
6. ECS deploys blue/green via CodeDeploy

Branch protections: feature branches → dev only; main/release → staging; production requires manual approval.

**Config & secrets**: non-sensitive config in repo files; secrets in SSM/Secrets Manager injected at runtime; least-privilege IAM throughout.

**Monitoring**: New Relic APM + CloudWatch, with Slack alerting.

**Security**: multi-account isolation, IAM + STS AssumeRole (no long-term keys), KMS encryption, tightly scoped Security Groups.

> [!TIP]
> **30-second elevator version:** Multi-account AWS setup, Terraform-provisioned infrastructure, Lambda (Serverless Framework) for backend services, ECS Fargate behind ALB for containers, GitLab CI/CD with STS AssumeRole for cross-account deploys, secrets in SSM/Secrets Manager, monitoring via New Relic and CloudWatch, blue/green deploys for ECS and rolling deploys for Lambda — secure, scalable, fully automated releases.
</details>

---

<div align="center">

⭐ *If this helped you prep, consider starring the repo.*

</div>
