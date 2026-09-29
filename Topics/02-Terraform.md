# Senior DevOps Interview Questions: Terraform

### Q: How do you import existing, manually created cloud infrastructure into Terraform state management?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is `terraform import`, but it's important to know it only updates the state file — it doesn't write the matching code for you.

What I normally do is write the resource block in code first, run the import, and then run `terraform plan` right away to check if anything looks different.

If I skip that last check, Terraform might try to delete or change the exact resource I just imported, because the code doesn't fully match what's actually there yet.

</details>

---

### Q: How do you structure Terraform configurations to manage multiple environments (Dev, Staging, Prod) without code duplication?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I write one shared module for each piece of infrastructure, like a VPC or a database.

Then each environment — dev, staging, prod — has its own small folder that calls that same module with its own settings, and its own separate state file.

I prefer this over Terraform Workspaces, because the separation here is real. There's no shared state file where picking the wrong option by mistake could end up touching production.

</details>

---

### Q: How do you configure a secure Terraform remote backend, and how do you recover if a state file is accidentally deleted?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I store the state file in S3, with encryption and versioning turned on, plus a lock so two people can't run `apply` at the same time and step on each other.

If someone deletes the state file by mistake, I just restore the last good version from S3's history, which usually takes a couple of minutes.

That's exactly why I always make sure versioning is enabled — without it, if the file is lost, I'd have to check every real resource by hand and rebuild the state piece by piece, which is a much longer and riskier job.

</details>

---

### Q: How do Local Exec and Remote Exec provisioners function in Terraform, and when should you use them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I only reach for these as a last resort.

`local-exec` runs a command on my own machine after a resource is created. `remote-exec` connects into the new server and runs a command there.

The problem is Terraform doesn't really track what either of these actually do — they can fail quietly, and running them again doesn't always fix it. What I normally do instead is set up servers using a startup script or a pre-built image, since those are more reliable and easier to repeat.

</details>

---

### Q: How do you manage sensitive credentials and database passwords securely in Terraform configurations?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Secrets never go into the code, and they never get committed to Git.

If a value is sensitive, I mark it with `sensitive = true`, so Terraform hides it from the screen and from logs. For something like a database password, I pull it in from a secrets manager at run time instead of typing it in directly.

One thing worth knowing — `sensitive = true` only hides the value on screen. It doesn't remove it from the state file, the real value is still sitting there. So the state file itself also needs to be locked down and encrypted.

</details>

---

### Q: What's the difference between `count` and `for_each` in Terraform, and why does it matter when resources change?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`count` tracks resources by their position in a list — first one, second one, and so on.

The problem is if that list order ever changes, Terraform thinks a different resource is now sitting at that position, and it tries to delete and recreate things that didn't actually need to change.

`for_each` tracks resources by a real key, like a name, instead of a position, so the same list reordering doesn't cause any unnecessary changes. I default to `for_each` now for anything where the list of items might change over time.

</details>

---

### Q: How do you safely upgrade a Terraform provider version without breaking existing infrastructure?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't let the provider version float freely. I pin it to a specific version and commit the lock file to Git, so every pipeline uses exactly the same version.

When I do want to upgrade, I bump the version on a branch, run it against a non-production environment first, and read the plan output carefully before applying anything.

Some upgrades quietly change how a resource behaves, not just the version number, so I never assume an upgrade is safe just because it installed cleanly.

</details>

---

### Q: How do you handle a Terraform state file that's grown huge and makes every plan slow?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A giant state file is usually a sign that too much infrastructure is being managed in one place. Every plan has to check every single resource in that file, so the bigger it gets, the slower everything gets, even for a tiny change.

What I normally do is split it up — separate state files for separate layers, like networking, databases, and applications, each managed on its own.

If one layer needs information from another, I read it through a safe, read-only reference instead of putting everything in one giant file for convenience.

</details>

---

### Q: Your Terraform pipeline fails because `terraform plan` can't get a lock on the state file — it says another process already has it. How do you fix this safely?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The lock exists on purpose, to stop two applies running at the same time and corrupting the state, so I never force past it without checking first.

First I check who's actually holding the lock — the DynamoDB lock table entry usually shows which pipeline run or machine grabbed it. Then I check if that job is actually still running, or if it crashed without cleaning up.

If it's genuinely dead, I can safely force-unlock using the lock ID from the error, but I need to be completely sure that job isn't mid-apply somewhere, since force-unlocking while a real apply is in progress is exactly how you corrupt the state. If it's a teammate's job, I just message them first.

</details>

---
