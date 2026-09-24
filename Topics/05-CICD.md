# Senior DevOps Interview Questions: CI/CD & Git

### Q: How do you structure an automated CI/CD pipeline for infrastructure provisioning using Terraform?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I split the pipeline into clear steps. On every pull request, it checks the code formatting, runs a validation check, does a quick security scan, and then builds a plan. That plan gets posted right on the pull request, so the team can review it before anything real happens. Once it's approved and merged, the apply step runs. It doesn't build a new plan at that point — it uses that exact same plan file that was already reviewed. That way, what got approved is exactly what runs. Production is always behind a manual approval step too.

</details>

---

### Q: How do you manage feature development and release workflows using Git branching strategies, Pull Requests, and Code Reviews?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I keep feature branches short and merge them through pull requests, instead of using long branches that sit around for weeks and drift out of sync. Every pull request has to pass automated tests, and also needs a real review from a teammate before it can merge. Once it's approved, the pipeline picks it up and moves it toward production on its own, no extra manual steps needed. In the review itself, I focus on things a tool can't check, like whether the actual approach makes sense, not just small style issues.

</details>

---

### Q: What is the difference between Git Merge and Git Rebase, and when should a team prefer a linear commit history?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git merge` just adds a new commit and keeps the real history of both branches. It's always safe, nothing gets changed. `git rebase` replays your commits on top of the latest code, which gives you a cleaner, straight-line history, but every commit gets a new ID in the process. I use rebase to update my own branch before opening a pull request. But I never rebase a branch that other people have already pulled, because it breaks their copy of it.

</details>

---

### Q: How do you resolve Git merge conflicts manually during branch integration?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

When Git can't automatically combine two changes to the same lines, it stops and marks the file so I can see exactly where the conflict is. My job is to open that file, look at both versions, pick what's actually correct, and remove the markers. Then I mark the file as fixed, and continue the merge. If things get too messy, I can cancel the whole thing and go back to where I started, which is a good safety net before a tricky conflict.

</details>

---

### Q: How are Git Tags and GitHub Releases utilized to manage immutable deployment versions?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use a real version tag, like `v2.1.0`, to mark an actual release point in the code. Pushing that tag is what kicks off the release pipeline — it builds everything and tags the final image with that same version number, never something generic like "latest." That part matters a lot, because deploying something called "latest" means you can never be fully sure what's actually running. With a real version number, I can always trace what's live in production straight back to the exact code it came from.

</details>

---

### Q: How do you keep secrets, like API keys and passwords, safe inside a CI/CD pipeline?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never sit in the pipeline config file itself, and never get printed in build logs. I store them in the CI tool's own secret storage, or better, pull them from a real secrets manager at run time, so the pipeline only ever holds a short-lived reference, not the actual value. I also make sure secrets used for one environment, like staging, can't be read by a pipeline running against a different environment, like production. And I turn on masking, so even if a secret accidentally gets printed somewhere, the actual value is hidden in the log output.

</details>

---

### Q: How would you add automatic security scanning into a CI/CD pipeline without slowing developers down too much?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I add scanning at two points. Code scanning runs early, right when a pull request opens, checking the source code itself for common mistakes and bad patterns — that's fast, so it doesn't slow anyone down. Then, after the build step, I scan the actual built image for known vulnerabilities. I only fail the pipeline on serious, high-severity issues at first, not every small warning, since blocking on everything just trains people to ignore the results. Once the team trusts the scanner and the noise is low, I tighten the rules over time.

</details>

---

### Q: How do you design rollback so that a bad deployment can be undone quickly, without a lot of manual steps?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I make sure every deployment is tied to one clear version, so rolling back just means redeploying the last known-good version, not trying to manually undo individual changes. I keep the previous version's artifact ready and available, not deleted right after a new deploy, so it's there instantly if needed. For the riskiest changes, I'll also add an automatic rollback trigger — if error rates spike right after a deploy, the pipeline rolls back on its own, instead of waiting for a person to notice and react, which is usually the slowest part of any incident.

</details>

---

### Q: You do a blue/green deployment, the new environment passes its health checks and gets traffic, but a few minutes later 5xx errors spike and CPU is high on the new side. How do you roll back?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is exactly why blue/green is worth the setup cost — the old environment is still sitting there, fully working, so I don't have to rebuild anything to recover. My first move is just triggering the rollback, which sends traffic straight back to the old, healthy environment. That's usually done in a couple minutes, and it stops the user-facing pain immediately.

Only once traffic is stable do I actually go dig into why the new version failed. I'd pull the logs from the failed instances and look for real errors — maybe a new call to some other service that's failing, or a bad config that only shows up under real traffic. I'd also take a hard look at what the health check was actually checking — a lot of the time it's just confirming the web server responds, not that the app is actually working end to end. A better health check, one that actually tests a real internal dependency, would've caught this before the full switch happened. Once the real bug's fixed, I retry the deployment, I don't just push the same broken version again.

</details>

---
