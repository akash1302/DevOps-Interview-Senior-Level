<div align="center">

# DAY 3 — TERRAFORM SCENARIO-BASED INTERVIEW QUESTIONS
## Senior DevOps / Platform Engineer — Interview Preparation

Quick, plain-English candidate answers — short and to the point, the way you'd actually say it out loud in an interview.

</div>

---

## Table of Contents

1. [Migrating State to a New Backend](#1-migrating-state-to-a-new-backend)
2. [`count` vs `for_each` — Resource Reordering Risk](#2-count-vs-for_each--resource-reordering-risk)
3. [`taint` and Forcing a Resource Replacement](#3-taint-and-forcing-a-resource-replacement)
4. [`terraform apply -target` — When It's Safe](#4-terraform-apply--target--when-its-safe)
5. [Workspaces vs Separate State Files](#5-workspaces-vs-separate-state-files)
6. [Provider Version Pinning and Upgrade Risk](#6-provider-version-pinning-and-upgrade-risk)
7. [State File Performance at Scale](#7-state-file-performance-at-scale)
8. [Circular Dependency Between Two Resources](#8-circular-dependency-between-two-resources)
9. [Policy-as-Code Gates Before Apply](#9-policy-as-code-gates-before-apply)
10. [Private Module Registry Authentication in CI](#10-private-module-registry-authentication-in-ci)

---

### Q: Your team needs to migrate Terraform state from a legacy local backend to S3, for a stack that's already running in production. How do you do it without downtime or losing track of resources?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never touch production resources during this, it's purely a state file move. First, I define the new S3 backend block in the code, then run `terraform init -migrate-state`, and Terraform copies the existing state over and asks for confirmation before switching. I always back up the old state file first, just in case. Once it's migrated, I run a `plan` immediately to confirm it shows zero changes, which proves nothing actually drifted during the move.

</details>

---

### Q: A team member changes a `count`-based resource list from a fixed list to a filtered one, and now Terraform wants to destroy and recreate half the resources that didn't actually change. What happened, and how do you prevent it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`count` tracks resources by their position in the list, like index 0, 1, 2 — it has no idea what the actual value at that position is. So if the list order shifts at all, Terraform thinks the resource at that index is now something different, and it destroys and recreates it, even though the same value still exists somewhere else in the list. The fix is using `for_each` with a map or a set instead, since that tracks resources by a stable key, not by position. I always default to `for_each` now unless I have a real reason not to.

</details>

---

### Q: A resource is stuck in a broken state — the actual AWS resource is fine, but Terraform's state for it is corrupted or out of sync. How do you force Terraform to recreate it cleanly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

That's exactly what `terraform taint` is for, or on newer versions, `terraform apply -replace=<resource_address>`. It marks the resource so the next `apply` destroys and recreates it, without touching anything else in the plan. I always run `plan` right after to confirm only that one resource shows up as changing, since tainting the wrong address can quietly catch other resources in a dependency chain.

</details>

---

### Q: When is it actually safe to use `terraform apply -target`, and why do senior engineers usually avoid it as a habit?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`-target` applies just one resource and skips the rest of the plan, which sounds convenient but it's risky as a habit. The big catch is it can leave your state out of sync with your actual desired config, since dependent resources don't get reconciled at the same time. I only use it in a real emergency, like fixing one broken resource fast during an incident, and I always follow it up with a full, untargeted `apply` right after to bring everything back in sync.

</details>

---

### Q: Your team is debating using Terraform workspaces versus separate directories with separate state files for dev, staging, and prod. What's your actual recommendation and why?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Workspaces share the same backend and the same code, they just swap out a variable, which sounds simple but makes it easy to accidentally run against the wrong environment if you forget which workspace you're on. In my experience, separate directories with separate backends and separate state files are much safer for real environments like prod, because the isolation is structural, not just a flag you have to remember to check. I'd only use workspaces for short-lived, throwaway environments, like a per-feature-branch preview stack.

</details>

---

### Q: A provider upgrade silently changes the behavior of a resource, and a routine `apply` in CI ends up modifying production unexpectedly. How do you prevent this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This happens when provider versions aren't pinned tightly, so CI just pulls in whatever's newest at apply time. I always pin exact provider versions in the `required_providers` block, and commit the `.terraform.lock.hcl` file to Git, so every environment resolves the exact same version. Upgrades are a deliberate, reviewed step — bump the version, run `plan` in a non-prod environment first, and check the diff carefully before it ever touches prod.

</details>

---

### Q: A single Terraform state file has grown to thousands of resources, and `plan` now takes several minutes even for a one-line change. How do you fix this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A giant state file is almost always a sign the stack needs to be split up. Every `plan` has to refresh every resource in that file, so the bigger it gets, the slower everything gets, even for tiny changes. I break it into smaller, logically separate stacks — like network, compute, and data — each with its own state file, connected through `terraform_remote_state` where needed. That keeps each `plan` fast and also shrinks the blast radius of any one change.

</details>

---

### Q: Two resources in your Terraform code seem to depend on each other — Terraform throws a circular dependency error. How do you actually resolve that?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Terraform builds a dependency graph from your code, and a true circular reference, like A needing B's output and B needing A's output, just can't be resolved automatically. Usually this means the resources are genuinely too tightly coupled and need to be restructured, or one side of the dependency can be broken using a separate resource, like an attachment resource instead of an inline reference. I look at the real relationship first, since forcing it with tricks like `depends_on` alone doesn't fix a genuine cycle.

</details>

---

### Q: How do you make sure a Terraform plan that violates a security policy, like an open security group or an unencrypted S3 bucket, never actually reaches `apply`?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't rely on someone catching it in a manual review, since that's easy to miss. I run a policy-as-code tool, like `tfsec`, `checkov`, or Sentinel if we're on Terraform Cloud, as a required step in the pipeline before `plan` even gets approved. If it flags a violation, the pipeline just fails outright and blocks the merge. That way the check is automatic and consistent, not dependent on someone remembering to look for it.

</details>

---

### Q: Your Terraform modules live in a private module registry, and CI needs to pull them during `init`. How do you authenticate that without hardcoding a token anywhere?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never hardcode a token in `.terraformrc` or commit it anywhere. Instead, CI pulls a short-lived credential at runtime, usually through the same OIDC-based role assumption we use for AWS access, and writes it into the CLI config just for that run. The token only exists for the length of the pipeline job and is scoped to read access on the registry, nothing more. That way there's no long-lived secret sitting around that could leak.

</details>

---

<div align="center">

⭐ *If this helped you prep, consider starring the repo.*

</div>
