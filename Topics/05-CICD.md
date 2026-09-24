# Senior DevOps Interview Questions: CI/CD & Git

### Q: How do you structure an automated CI/CD pipeline for infrastructure provisioning using Terraform?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I break the Terraform pipeline into clearly separated stages instead of one big script. On every pull request, it runs `terraform fmt -check` and `terraform validate`, then a security scan with something like `checkov`, and finally generates a `terraform plan -out=tfplan`, saved as a pipeline artifact and posted directly as a PR comment for the team to actually review before anything touches real infrastructure.

Once that's merged to `main`, the apply stage is gated behind manual approval — it doesn't run a fresh plan at that point, it applies the exact `tfplan` artifact that was already reviewed, which guarantees what got approved is exactly what executes, with nothing able to drift in between. In GitLab CI that's four distinct stages: `fmt-validate`, `security-scan`, `plan` triggered on PR, and `apply` set to `when: manual` and scoped to `main`. That separation is what actually prevents an accidental, unreviewed change from ever reaching production infrastructure.

</details>

---

### Q: How do you manage feature development and release workflows using Git branching strategies, Pull Requests, and Code Reviews?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I push for short-lived feature branches off `main`, merged back through Pull Requests, rather than long-lived release branches that just accumulate drift over time. Every PR triggers CI — tests, linters, security scans — and on top of the automated checks, it needs at least one real peer review before it can merge.

Concretely, that means branch protection on `main` requiring a PR with at least one approval, dismissing stale approvals automatically if new commits land after the review, requiring the CI status checks to actually pass, and enforcing linear history so the log stays clean and bisectable. Once something's approved and merged, the CD pipeline picks it up automatically and pushes it toward staging and then production — there's no separate manual "release branch" step slowing things down. Code review itself is where I focus on the stuff automation can't catch — architecture decisions, security implications, whether the tests actually cover the real risk, not just style nits.

</details>

---

### Q: What is the difference between Git Merge and Git Rebase, and when should a team prefer a linear commit history?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git merge` creates a new commit that ties two branches together and keeps the true chronological history intact — it's always safe, because it never rewrites commits anyone else might already have. `git rebase` replays your commits on top of a new base, which gives you clean, linear history, but every replayed commit gets a brand new hash — that's a real history rewrite.

In practice, I rebase my own feature branch locally to pull in the latest `main` before opening a PR — `git checkout feature/login`, `git fetch origin`, `git rebase origin/main`, resolve anything that conflicts, then push with `git push --force-with-lease` instead of a plain force, since that protects against overwriting a teammate's work I haven't seen yet. But I never rebase a branch that's already shared or public — that's the golden rule, because rewriting shared history breaks everyone else's local copy of it. Linear history really pays off when you're using `git bisect` to hunt down a regression — a clean, linear log makes that binary search actually trustworthy.

</details>

---

### Q: How do you resolve Git merge conflicts manually during branch integration?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

When Git can't automatically reconcile two branches — usually because the same lines got changed independently on both sides — it stops and marks the conflicted file with `<<<<<<<`, `=======`, and `>>>>>>>` markers. My job at that point is to open the file, actually understand both versions, decide which lines are correct, or blend them, and remove the markers entirely.

Say a config file shows `server_port = 8080` on one side and `server_port = 9090` on the other — I'd manually pick the right value, clean out the markers, then run `git add server.conf` to mark it resolved, and finish with `git merge --continue` or `git rebase --continue` depending on which operation I was in the middle of. If things get messy enough that I want to bail entirely, `git merge --abort` cleanly puts the repo back to exactly where it was before the merge started, which is a good safety net to know about before diving into a gnarly multi-file conflict.

</details>

---

### Q: How are Git Tags and GitHub Releases utilized to manage immutable deployment versions?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use annotated tags following semantic versioning — something like `v2.1.0` — to mark an actual release point in history. An annotated tag, created with `git tag -a v2.1.0 -m "Release version 2.1.0 with payment gateway integration"`, carries real metadata: who tagged it, when, and why, which a lightweight tag doesn't give you.

Pushing that tag is what kicks off the release pipeline — the CI config just watches for a pattern like `v*.*.*` on push, builds the artifacts, and tags the Docker image with the actual version number, `1.2.0`, never `latest`. That last part matters a lot — deploying `latest` to production means you can never be sure which code is actually running, and rollback becomes guesswork. With a real version tag on the image, I can trace a running container straight back to the exact Git commit it came from, and the GitHub Release entry attached to that tag gives the team a clear, permanent changelog for that version.

</details>

---
