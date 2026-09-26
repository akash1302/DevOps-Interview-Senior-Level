# Senior DevOps Interview Questions: Git & GitHub/GitLab

### Q: A teammate force-pushed to `main` and overwrote several commits. How do you recover?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, I stop anyone else from pushing to `main` until this is sorted, since more activity on top just makes recovery harder.

Then I look for the old commit hash. `git reflog` on anyone's machine that had `main` checked out recently usually still has it, even though it's gone from the normal log. If not, GitHub/GitLab keeps the commit objects around for a while, and the old SHA is often still visible in a merged PR's history.

Once I have the hash, it's a straight `git reset --hard <old-sha>` on `main`, then push.

*Find the old SHA via reflog or PR history → reset main to it → push → turn on branch protection so it can't happen again.*

The actual fix afterward is enabling branch protection with force-push disabled on `main`, so this specific mistake becomes structurally impossible.

</details>

---

### Q: You need to revert a bad merge that's already been deployed to production. Walk through it.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use `git revert`, never a hard reset — a reset rewrites history other people have already pulled, and it can confuse a pipeline that's watching `main` for new commits.

A merge commit has two parents, so I revert with `git revert -m 1 <merge-sha>`, telling Git to go back to the state before the merge while keeping mainline history intact. I always double-check with `git log --graph` first, since reverting against the wrong parent silently keeps the bug instead of removing it.

*Confirm mainline parent → `git revert -m 1 <sha>` → push → pipeline deploys the revert like any normal commit → confirm in production → open a separate PR with the real fix.*

Once pushed, the revert commit flows through the normal deploy pipeline like any other change, and I open a fresh PR for the actual fix instead of trying to force the original branch to work under pressure.

</details>

---

### Q: How do you handle a massive merge conflict across 50+ files after a long-lived feature branch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A conflict this size is really a process failure — a branch that old should've been rebasing against `main` regularly the whole time.

For the immediate fix, I use `git rebase main` instead of a merge, since rebase replays commits one at a time, so I'm resolving small conflicts per commit instead of one giant tangle. I also turn on `git config rerere.enabled true` first, so if the same conflict pattern repeats across multiple commits — usually a shared config file — Git auto-applies the resolution after the first time.

If a lockfile is causing most of the noise, I resolve the real code conflicts first, then just regenerate the lockfile fresh instead of hand-merging it.

Going forward, I push for shorter branches and a required rebase against `main` at least every couple of days.

</details>

---

### Q: Your GitHub Actions workflow is exposing secrets in the logs. How do you find and fix it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

GitHub auto-masks a secret referenced through `${{ secrets.X }}`, so if it's showing up in plain text, it's usually a transformed version of it — base64-encoded, JSON-stringified, concatenated into another string — since masking only catches the exact literal value.

I find the exact command producing the leak, fix it, and treat the secret as compromised the moment it appeared, even in a private repo — rotate it immediately, don't wait.

*Find the command leaking it → confirm it's a derived/transformed value → fix the script → rotate the secret → mask any derived value going forward with `::add-mask::`.*

For anything computed from a secret at runtime, I explicitly mask it with `echo "::add-mask::$derived_value"`, since the default masking only covers the raw secret value, not anything built from it.

</details>

---

### Q: How do you set up branch protection so nobody can bypass code review, even admins?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The setting most teams miss is "include administrators" — without it, repo admins can bypass every other protection rule, and I've seen teams assume they were protected when they weren't.

What I actually turn on: require a PR with at least one approval, require status checks to pass, require the branch to be up to date before merging, and include administrators in all of it. For sensitive paths, I add a `CODEOWNERS` file so a specific team has to sign off on changes there specifically.

On GitLab, it's protected branches plus merge request approval rules, with force-push explicitly disabled — that's a separate toggle from approvals.

I always confirm it actually works by trying to push directly to the branch myself as an admin, rather than assuming the settings did what I expected.

</details>

---

### Q: A developer accidentally committed a large binary file or a credentials file to the repo. How do you remove it from history?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Deleting the file in a new commit doesn't fix this — it's still sitting in every earlier commit, and anyone can check out an old commit and pull it right back out.

If it's a credential, step one before touching Git at all is rotating it. A leaked key is burned the moment it was committed, regardless of whether the repo is private.

For history itself, I use `git filter-repo` to strip that file path out of every commit. That rewrites every commit hash after it was introduced, so it needs real coordination — everyone pushes in-progress work first, then re-clones after the force-push instead of trying to merge their old clone against the new history.

*Rotate the leaked credential first → run `git filter-repo` to strip the file from history → force-push → team re-clones → add a pre-commit secret scanner like gitleaks so it can't happen again.*

</details>

---

### Q: GitLab CI pipeline is stuck showing "pending" and never starts. How do you troubleshoot?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

"Pending" means the job is queued waiting for a runner — it hasn't failed, so I check runner availability first, not the pipeline config.

The most common cause is a `tags:` mismatch — the job requests a runner tag, like `gpu`, that no registered runner actually has. The job just sits pending forever with no error, since GitLab is correctly waiting for a runner that will never show up.

If tags match, I check the runner's real health directly, not just its reported status — I've had a runner show "online" while its Docker daemon was actually down, so it couldn't execute anything despite looking healthy.

I also check if it's one project or the whole instance, since that tells me whether the problem is this pipeline's config or the runner fleet itself.

</details>

---
