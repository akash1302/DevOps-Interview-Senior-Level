# Senior DevOps Interview Questions: CI/CD & Git

### Q: How do you structure an automated CI/CD pipeline for infrastructure provisioning using Terraform?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I split the pipeline into clear steps instead of one big script. On every pull request, it checks formatting, runs `terraform validate`, does a quick security scan, and then generates a `plan` that gets posted right on the PR so the team can actually review it before anything real happens. Once that's approved and merged to `main`, the apply step doesn't generate a fresh plan — it applies that exact same plan file that was already reviewed. That way, what got approved is exactly what runs, nothing can quietly change in between. Production apply is always behind a manual approval step too.

</details>

---

### Q: How do you manage feature development and release workflows using Git branching strategies, Pull Requests, and Code Reviews?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I keep feature branches short-lived and merge them into `main` through pull requests, instead of long-lived release branches that just drift out of sync over time. Every PR has to pass automated checks — tests, linting, security scans — and also needs at least one real review from a teammate before it can merge. Once it's approved, the pipeline picks it up automatically and moves it toward staging and then production, no separate manual release step needed. In code review, I focus on the things automation can't catch, like whether the design actually makes sense, not just small style issues.

</details>

---

### Q: What is the difference between Git Merge and Git Rebase, and when should a team prefer a linear commit history?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git merge` just adds a new commit and keeps the real history of both branches — always safe, since nothing gets rewritten. `git rebase` replays your commits on top of a new base, giving cleaner, straight-line history, but every commit gets a brand new ID in the process. I use rebase to update my own branch with the latest `main` before opening a PR, then push with `--force-with-lease`, which protects against overwriting someone else's work I haven't seen yet. The one hard rule — never rebase a branch that other people already pulled, since it breaks their local copy.

</details>

---

### Q: How do you resolve Git merge conflicts manually during branch integration?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

When Git can't automatically combine two changes to the same lines, it stops and marks the file with conflict markers. My job is to open that file, look at both versions, decide what's actually correct, and remove the markers completely. Once it's fixed, I run `git add` on that file to mark it resolved, then continue the merge or rebase. If things get too messy, `git merge --abort` cleanly resets everything back to before the merge started, which is a good safety net before diving into a hard multi-file conflict.

</details>

---

### Q: How are Git Tags and GitHub Releases utilized to manage immutable deployment versions?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use annotated tags, like `v2.1.0`, to mark an actual release point. Pushing that tag is what kicks off the release pipeline — it builds the artifacts and tags the Docker image with that real version number, never `latest`. That last part matters a lot, since deploying `latest` means you can never be fully sure which code is actually running, and rolling back becomes guesswork. With a real version on the image, I can always trace exactly what's running in production straight back to the specific commit it came from.

</details>

---
