# Senior DevOps Interview Questions: CI/CD & Git

## Q1. How do you structure an automated CI/CD pipeline for infrastructure provisioning using Terraform?

### Answer
Automating Terraform in a CI/CD pipeline requires separating validation, planning, and execution stages with automated quality gates and manual approval steps. On code commit or Pull Request (PR), the CI pipeline executes syntax linting (`terraform fmt -check`, `tflint`), initialization (`terraform init`), validation (`terraform validate`), and plan generation (`terraform plan`). The plan output is posted as a PR comment for team review. Upon merging to the main branch, a manual approval gate triggers the execution stage (`terraform apply`), applying the exact pre-generated plan file.

### Interview Answer
"In GitLab CI or GitHub Actions, I break the Terraform pipeline into distinct stages. On every Pull Request, the pipeline runs `fmt`, `validate`, security scanning with `checkov`, and generates a `terraform plan` output file saved as an artifact. The plan is posted directly to the PR for peer review. Once merged to `main`, the pipeline requires a manual production approval button before executing `terraform apply` using the approved plan artifact, preventing accidental state changes."

### Practical Example
GitLab CI pipeline stages definition (`.gitlab-ci.yml`):
1. `fmt-validate`: Runs `terraform fmt` and `terraform validate`.
2. `security-scan`: Runs `tfsec` or `checkov`.
3. `plan`: Executes `terraform plan -out=tfplan` (triggers on PR).
4. `apply`: Executes `terraform apply tfplan` (runs on `main` branch with `when: manual`).

### Follow-up Questions
* Why should you pass a pre-generated plan file (`tfplan`) to `terraform apply` in automated pipelines?
* How do you securely handle cloud authentication credentials inside CI/CD runners (e.g., OIDC vs static keys)?
* What automated testing tools (e.g., `terratest`) can be integrated into the CI pipeline?

### Key Points
* PRs trigger automated formatting, validation, security scanning, and plan generation.
* Pre-generated plan artifacts ensure that the executed changes match the reviewed plan exactly.
* Production apply stages must enforce branch protection rules and manual approval gates.

---

## Q2. How do you manage feature development and release workflows using Git branching strategies, Pull Requests, and Code Reviews?

### Answer
Modern DevOps teams use Trunk-Based Development or GitHub Flow to maintain high deployment velocity. Developers create short-lived feature branches off the main branch (`main`). All code modifications are submitted back via Pull Requests (PRs) / Merge Requests (MRs). PRs trigger automated CI status checks (unit tests, linters, security scans) and require peer code reviews before merging. Peer reviews validate architectural decisions, security practices, and maintainability, acting as a human quality gate before automated CD triggers deployment.

### Interview Answer
"I advocate for short-lived feature branches merging into `main` via Pull Requests. We enforce branch protection rules on `main` requiring at least one peer code review approval and successful CI status check passes (tests, security scans, build checks). Code reviews focus on design, security, and test coverage. Once approved and merged, our CD pipeline automatically triggers deployments to staging and production, avoiding long-lived stale release branches."

### Practical Example
GitHub repository settings configured with Branch Protection Rules on `main`:
* Require a pull request before merging (minimum 1 approval).
* Dismiss stale pull request approvals when new commits are pushed.
* Require status checks to pass before merging (`build`, `lint`, `security-scan`).
* Require linear commit history.

### Follow-up Questions
* How does Trunk-Based Development differ from traditional GitFlow in high-velocity CI/CD teams?
* How do feature flags allow merging code to `main` continuously without exposing unreleased features?
* What strategies resolve pull request stale branch drift before merging?

### Key Points
* Short-lived feature branches minimize merge complexity and code drift.
* Branch protection rules enforce required CI status checks and peer review approvals.
* Automated CD pipelines trigger immediately upon merging approved PRs into the primary branch.

---

## Q3. What is the difference between Git Merge and Git Rebase, and when should a team prefer a linear commit history?

### Answer
`git merge` integrates changes from a source branch into a target branch by creating a new non-fast-forward "merge commit", preserving the true chronological history and branch topology. `git rebase` reapplies commits from the feature branch individually on top of the target branch's tip, creating new commit hashes and resulting in a clean, linear history without extra merge commits. Rebase is preferred for keeping feature branches up-to-date with `main`, but should never be executed on shared public branches (`golden rule of rebase`).

### Interview Answer
"I use `git rebase` locally on my feature branch to pull in the latest changes from `main` before submitting a PR. This keeps the commit history completely linear and easy to audit with `git log` or `git bisect`. However, I follow the golden rule of rebase: never rebase public or shared branches like `main` because rewriting shared history corrupts commit hashes for other team members. For final PR merges into `main`, we use squash-and-merge or linear rebase."

### Practical Example
Updating a feature branch with latest `main` changes:
```bash
git checkout feature/login
git fetch origin
git rebase origin/main
# Resolve any conflicts commit-by-commit, then:
git push --force-with-lease origin feature/login
```

### Follow-up Questions
* Why is `git push --force-with-lease` safer than `git push --force` after rebasing a branch?
* How does a linear commit history simplify debugging using `git bisect`?
* What is "Squash and Merge" and what problem does it solve in Git log histories?

### Key Points
* `git merge` preserves true branch topology by creating a dedicated merge commit.
* `git rebase` rewrites commit history to create a clean, linear sequence of commits.
* Never rebase shared public branches to avoid breaking team commit references.

---

## Q4. How do you resolve Git merge conflicts manually during branch integration?

### Answer
A Git merge conflict occurs when Git cannot automatically reconcile differences between two branches—typically when the same lines of code in a file were modified independently in both branches. To resolve conflicts manually, Git marks the conflicted files with conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`). The engineer must inspect the marked files, choose the correct lines of code to keep, delete the conflict markers, mark the files as resolved using `git add`, and finalize the merge/rebase commit using `git merge --continue` or `git rebase --continue`.

### Interview Answer
"When a merge conflict occurs, Git halts the process and annotates the conflicting files. I open the affected files, identify the HEAD changes versus the incoming branch changes demarcated by conflict markers, and manually edit the code to preserve the correct logic. After editing, I run `git add <file>` to stage the resolution, and then execute `git merge --continue` or `git rebase --continue` to finish integrating the branches."

### Practical Example
Conflicted file content:
```text
<<<<<<< HEAD
server_port = 8080
=======
server_port = 9090
>>>>>>> feature/port-update
```
Resolution process:
1. Manually edit file to keep `server_port = 9090` and remove markers.
2. Stage file: `git add server.conf`
3. Complete operation: `git merge --continue`

### Follow-up Questions
* What is the difference between `git merge --abort` and `git rebase --abort`?
* How does setting `git config rerere.enabled true` (Reuse Recorded Resolution) help resolve recurring conflicts?
* How do IDE conflict resolution tools assist during complex multi-file conflicts?

### Key Points
* Conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) isolate conflicting code blocks.
* Resolution requires manual editing, removing markers, staging (`git add`), and continuing the commit.
* `git merge --abort` safely returns the repository to its pre-merge state if errors occur.

---

## Q5. How are Git Tags and GitHub Releases utilized to manage immutable deployment versions?

### Answer
Git Tags create explicit pointers to specific commits in Git history, typically marking software release milestones (e.g., `v1.2.0`). Annotating tags (`git tag -a`) stores metadata including tagger identity, date, and release notes. In CI/CD pipelines, pushing a tag matching a version pattern (`v*.*.*`) automatically triggers release workflows—building immutable Docker images tagged with the semantic version, compiling binary artifacts, and publishing a GitHub Release entry with release release logs.

### Interview Answer
"I use annotated Git tags following Semantic Versioning (`v1.2.0`) to mark immutable release releases. When a tag is pushed, our CI pipeline intercepts the tag event, builds production artifacts, tags the Docker container image with `1.2.0` (avoiding mutable tags like `latest`), and attaches release release notes to GitHub Releases. This ensures complete traceability from the running container back to the exact Git commit."

### Practical Example
Creating and pushing an annotated release tag:
```bash
git tag -a v2.1.0 -m "Release version 2.1.0 with payment gateway integration"
git push origin v2.1.0
```
CI pipeline trigger rule:
```yaml
on:
  push:
    tags:
      - 'v*.*.*'
```

### Follow-up Questions
* What is the difference between Lightweight tags and Annotated tags in Git?
* Why is deploying Docker containers tagged as `latest` considered a bad security and operational practice?
* What principles define Semantic Versioning (`MAJOR.MINOR.PATCH`)?

### Key Points
* Annotated Git tags create immutable versioned milestones in source control history.
* Tag pushes trigger CI pipelines to build matching versioned artifacts (Docker images, binaries).
* Never deploy mutable tags like `latest` to production environments.


---
